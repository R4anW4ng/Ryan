pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        // If Jenkins itself crashes mid-build, the default durability
        // setting tries to RESUME that build on next startup by
        // reconnecting to its old process handles. Since the whole
        // container (and everything in it) is gone by then, that resume
        // attempt is reconnecting to something that no longer exists —
        // which is the likely cause of Jenkins crashing again shortly
        // after every restart. This setting tells Jenkins to just mark
        // an interrupted build as failed on restart instead of trying
        // (and failing) to resume it.
        durabilityHint('PERFORMANCE_OPTIMIZED')
    }

    // NOTE: no pollSCM trigger here anymore. Under Multibranch Pipeline,
    // polling/branch-discovery is configured at the JOB level instead
    // ("Scan Repository Triggers -> Periodically", e.g. every 5 minutes),
    // since Multibranch also needs to detect new/deleted branches, which
    // a Jenkinsfile-level pollSCM trigger can't do on its own. Keeping
    // both would just mean two redundant polling mechanisms.

    environment {
        APP_NAME          = 'foodgorilla'
        POSTGRES_USER     = 'foodgorilla_admin'
        POSTGRES_PASSWORD = credentials('nutritrack-db-password') 
        POSTGRES_DB       = 'foodgorilla_db'
        
        FLASK_APP         = 'app.main'
        FLASK_ENV         = 'production'
        PYTHONPATH        = '/app'

        // Explicitly naming ONLY the base file disables Compose's automatic
        // merge of docker-compose.override.yml, so the test stack never
        // inherits host port bindings (see docker-compose.override.yml).
        COMPOSE_TEST_FILES = '-f docker-compose.yml'

        // Forces Compose to build images ONE AT A TIME instead of all
        // concurrently via BuildKit. This Codespace has no swap configured
        // (and can't have swap added — containers can't call swapon), so
        // building database+backend+frontend simultaneously, on top of the
        // Jenkins JVM and Docker daemon, previously spiked memory enough
        // to kill every running container at once. Sequential builds take
        // a bit longer but keep peak memory well below that ceiling.
        COMPOSE_PARALLEL_LIMIT = '1'
    }

    stages {
        // STAGE 1: CLEAN WORKSPACE & FETCH CODE
        stage('Checkout Code') {
            steps {
                echo 'Purging host workspace folder caches entirely...'
                deleteDir() 
                checkout scm
            }
        }

        // STAGE 1.5: HOST PREFLIGHT CHECKS (runs for every branch, before any build)
        stage('Preflight Checks via Ansible') {
            steps {
                echo '🔍 Verifying host prerequisites (Docker, Compose, disk space)...'
                // Read-only checks only — never touches docker compose up/down,
                // never interacts with the running app stack. Runs before any
                // build so a bad host environment fails fast and clearly.
                sh 'ansible-playbook -i "localhost," ansible/playbook.yml --tags preflight'
            }
        }

        // STAGE 2: ISOLATED TESTING (Runs first!)
        stage('Integration Testing') {
            steps {
                echo '🧹 DEFENSIVE CLEANUP: Wiping any stale test containers...'
                // Using "-p ${APP_NAME}_test" guarantees this command ONLY touches test setups
                sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test down -v --remove-orphans || true'

                echo 'Building and starting the full dependency chain (database -> backend -> frontend)...'
                // Now that backend has its own HEALTHCHECK and frontend has
                // depends_on: backend: condition: service_healthy, Compose
                // itself brings services up in the correct order and BLOCKS
                // (then fails the whole command) if a dependency never
                // becomes healthy. This replaces the old manual staggering
                // of "up -d database frontend" -> "sleep 10" -> "up -d backend".
                //
                // Explicitly naming only the app services (never "jenkins")
                // here too — even under the separate "_test" project
                // namespace, an unqualified "up" would still spin up an
                // unnecessary throwaway jenkins container every single run.
                sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test up -d --build database frontend backend'

                echo 'Checking container status...'
                sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test ps'

                echo 'Verifying backend health endpoint content (not just container status)...'
                // By this point Docker's own HEALTHCHECK already confirmed the
                // backend responds with HTTP 200 before frontend was allowed to
                // start — no retry loop needed here anymore. This check instead
                // confirms the JSON body itself reports a real DB connection,
                // which the container-level HEALTHCHECK doesn't inspect.
                sh '''
                    docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test exec -T backend python -c "
import urllib.request, json, sys
res = urllib.request.urlopen('http://127.0.0.1:5000/health-check', timeout=5)
body = json.loads(res.read())
print('Backend response:', body)
if body.get('database_connectivity') != 'CONNECTED':
    print('!!! Backend responded but database is not actually connected !!!')
    sys.exit(1)
"
                '''
            }
            post {
                always {
                    echo '--- CAPTURING LOGS (post-attempt, for diagnostics) ---'
                    sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test logs backend > backend_debug.log 2>&1 || true'
                    sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test logs frontend > frontend_debug.log 2>&1 || true'
                    sh 'cat backend_debug.log || true'
                    sh 'cat frontend_debug.log || true'
                    archiveArtifacts artifacts: 'backend_debug.log,frontend_debug.log', allowEmptyArchive: true

                    echo 'Cleaning up isolated test architecture environment...'
                    sh 'docker compose ${COMPOSE_TEST_FILES} -p ${APP_NAME}_test down -v --remove-orphans || true'
                }
            }
        }

        // ============ NEW: Security Scan stage — Ryan ============
        // Design choice: this scans the images the PREVIOUS stage just built,
        // not images pulled from a registry. This repo has no Docker Hub push
        // stage (and docker-compose.yml declares no "image:" keys), so there is
        // nothing published to pull — a registry-based scan could never go
        // green here. Integration Testing runs immediately above this on every
        // branch and every trigger, and its teardown is "down -v
        // --remove-orphans" WITHOUT --rmi, so the images it built are still
        // present on the Docker host when this stage runs.
        //
        // Compose names build-only images "<project>-<service>", and the test
        // project is "${APP_NAME}_test", so the two targets are
        // foodgorilla_test-backend:latest and foodgorilla_test-frontend:latest.
        //
        // Trade-off, stated plainly: we lose "re-scan the artifact that is
        // actually published" and gain "scan exactly what this commit builds."
        // Runs on EVERY trigger (push AND nightly) — this is the one stage
        // that intentionally has no TimerTrigger guard. The nightly run works
        // the same way: Integration Testing rebuilds both images from the
        // latest commit first, so the nightly scan re-scans a fresh build
        // against a freshly updated CVE database.
        //
        // Two scanners, two layers (defense in depth):
        //   - Trivy scans the built IMAGES: the base image's OS packages plus
        //     every library actually installed inside the container.
        //   - OWASP Dependency-Check scans the SOURCE manifests those images
        //     are built from (backend/requirements.txt,
        //     frontend/package-lock.json) against the NVD CVE database.
        // Install/config steps for both tools (locally + on this Jenkins
        // server): see SECURITY_SCANNING.md at the repo root.
        stage('Security Scan') {
            environment {
                // ---- Enforcement knobs (the ONLY lines to touch later) ----
                // Phase 1 (current): REPORT-ONLY, so day-one builds don't
                // hard-fail on pre-existing vulnerabilities nobody has
                // triaged yet. After the baseline cleanup
                // (SECURITY_SCANNING.md sections 6-7), flip to Phase 2:
                //   TRIVY_EXIT_CODE  '0'  -> '1'  (fail on HIGH/CRITICAL)
                //   DC_FAIL_CVSS     '11' -> '7'  (fail on CVSS >= 7.0;
                //                                  11 can never trigger — max is 10)
                TRIVY_EXIT_CODE = '0'
                TRIVY_SEVERITY  = 'HIGH,CRITICAL'
                DC_FAIL_CVSS    = '11'
            }
            steps {
                echo '🛡️ Scanning freshly built images for vulnerabilities (Trivy + OWASP DC)...'
                sh '''
                    # Preflight: fail fast with a fixable message instead of a
                    # cryptic one ten minutes into a scan.
                    command -v trivy >/dev/null 2>&1 || {
                        echo "ERROR: trivy is not installed on this Jenkins agent (SECURITY_SCANNING.md, section 5)."; exit 1; }
                    command -v dependency-check.sh >/dev/null 2>&1 || {
                        echo "ERROR: dependency-check.sh is not installed on this Jenkins agent (SECURITY_SCANNING.md, section 5)."; exit 1; }

                    # The two scan targets are built by Integration Testing
                    # immediately above. If they are missing, that stage did not
                    # build them under the expected name — say so here rather
                    # than letting Trivy fail with a bare "image not found".
                    BACKEND_IMAGE="${APP_NAME}_test-backend:latest"
                    FRONTEND_IMAGE="${APP_NAME}_test-frontend:latest"
                    for IMG in "${BACKEND_IMAGE}" "${FRONTEND_IMAGE}"; do
                        docker image inspect "${IMG}" >/dev/null 2>&1 || {
                            echo "ERROR: expected image ${IMG} is not present on this Docker host."
                            echo "It is built by the Integration Testing stage. Check that stage's"
                            echo "log, and see SECURITY_SCANNING.md section 9 (troubleshooting)."
                            exit 1; }
                    done

                    # Vulnerability DBs live on the jenkins_home volume so they
                    # survive container rebuilds — they can't live in the
                    # workspace because Checkout wipes it with deleteDir()
                    # every single build.
                    export TRIVY_CACHE_DIR="${JENKINS_HOME}/trivy-cache"

                    # Trivy image scans. Scan BOTH images before deciding
                    # pass/fail, so a failing backend scan never hides the
                    # frontend's findings (and vice versa).
                    SCAN_STATUS=0
                    trivy image \
                        --scanners vuln \
                        --severity ${TRIVY_SEVERITY} \
                        --exit-code ${TRIVY_EXIT_CODE} \
                        --no-progress \
                        --output trivy-backend.txt \
                        "${BACKEND_IMAGE}" || SCAN_STATUS=1
                    trivy image \
                        --scanners vuln \
                        --severity ${TRIVY_SEVERITY} \
                        --exit-code ${TRIVY_EXIT_CODE} \
                        --no-progress \
                        --output trivy-frontend.txt \
                        "${FRONTEND_IMAGE}" || SCAN_STATUS=1

                    echo "===== Trivy findings: backend image ====="
                    cat trivy-backend.txt || true
                    echo "===== Trivy findings: frontend image ====="
                    cat trivy-frontend.txt || true

                    # OWASP Dependency-Check over the source dependency
                    # manifests. --enableExperimental is required for the
                    # Python (requirements.txt) analyzers. NVD_API_KEY is
                    # optional but strongly recommended (SECURITY_SCANNING.md,
                    # section 4) — set +x so the key never lands in build logs.
                    set +x
                    NVD_KEY_ARG=""
                    if [ -n "${NVD_API_KEY:-}" ]; then
                        NVD_KEY_ARG="--nvdApiKey ${NVD_API_KEY}"
                        echo "Using NVD API key from Jenkins global configuration."
                    else
                        echo "WARNING: NVD_API_KEY is not set — NVD database updates will be slow/rate-limited (SECURITY_SCANNING.md, section 4)."
                    fi
                    dependency-check.sh \
                        --project "${APP_NAME}" \
                        --scan backend \
                        --scan frontend \
                        --exclude "**/node_modules/**" \
                        --enableExperimental \
                        --format HTML --format JSON \
                        --out dependency-check-report \
                        --data "${JENKINS_HOME}/dependency-check-data" \
                        --failOnCVSS ${DC_FAIL_CVSS} \
                        ${NVD_KEY_ARG} || SCAN_STATUS=1
                    set -x

                    # Non-zero only when a scanner found enforceable issues or
                    # genuinely errored — in report-only phase this stays 0
                    # unless a scan itself broke (which SHOULD fail the build).
                    exit ${SCAN_STATUS}
                '''
            }
            post {
                always {
                    // Reports for every run, pass or fail: build page ->
                    // "Archived artifacts" (trivy-*.txt + HTML/JSON DC report).
                    archiveArtifacts artifacts: 'trivy-*.txt, dependency-check-report/**', allowEmptyArchive: true
                }
            }
        }
        // ============ END NEW ============

        // STAGE 2.5: AUTO-OPEN PR TO MAIN (feature branches only, after tests pass)
        stage('Open Pull Request to main') {
            when {
                // Only feature/* branches reach here — main's own runs skip
                // this (main has nothing to open a PR against itself for),
                // and if Integration Testing failed, Jenkins never reaches
                // this stage at all, so a failed test run never opens a PR.
                branch pattern: 'feature/.*', comparator: 'REGEXP'
            }
            steps {
                echo '📬 Opening pull request into main...'
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        HTTP_CODE=$(curl -s -o response.json -w "%{http_code}" -X POST \
                            -H "Authorization: token $GH_TOKEN" \
                            -H "Accept: application/vnd.github+json" \
                            https://api.github.com/repos/Benedictusernametaken/Food-Gorilla/pulls \
                            -d "{\\"title\\":\\"Merge ${BRANCH_NAME} into main\\",\\"head\\":\\"${BRANCH_NAME}\\",\\"base\\":\\"main\\",\\"body\\":\\"Automated PR opened after Jenkins pipeline succeeded on ${BRANCH_NAME}.\\"}")

                        echo "GitHub API responded with HTTP $HTTP_CODE"
                        cat response.json

                        # 201 = PR created. 422 = a PR already exists for this
                        # branch (harmless — don't fail the build over it).
                        # Anything else is a genuine problem worth failing loudly for.
                        if [ "$HTTP_CODE" != "201" ] && [ "$HTTP_CODE" != "422" ]; then
                            echo "!!! Unexpected response while creating pull request !!!"
                            exit 1
                        fi
                    '''
                }
            }
        }

        // STAGE 3: PRODUCTION DEPLOYMENT (Only runs if Testing succeeds, AND only on main!)
        stage('Deploy to Production') {
            when {
                // Under Multibranch, every branch gets its own job. Without
                // this guard, a teammate's feature branch push would also
                // deploy straight to production — only merges to main should.
                branch 'main'
            }
            steps {
                echo 'Cleaning up and starting production services...'
                // No -f flags here, so Compose auto-merges docker-compose.yml +
                // docker-compose.override.yml and gets real host ports as intended.
                //
                // CRITICAL: explicitly name only the app services here.
                // A bare "docker compose up" with no service list would ALSO
                // bring up the "jenkins" service defined in this same compose
                // file — meaning a running Jenkins pipeline would try to
                // recreate/restart the very Jenkins container it's executing
                // inside of. Never remove this explicit service list.
                // Now that docker-compose.yml no longer hardcodes a global
                // "name:", Compose would otherwise derive a project name
                // from whatever folder this happens to run in — which for
                // Jenkins is its own internal workspace path, unrelated to
                // the actual app. Pinning "-p foodgorilla" here keeps
                // production deploys stable and predictable regardless of
                // that path, without reintroducing a name that silently
                // applies to every directory copy everywhere (the bug that
                // caused Jenkins to keep getting killed mid-pipeline).
                // CRITICAL FIX: "docker compose down" has NO way to scope to
                // specific services — it always tears down the ENTIRE named
                // project, jenkins included, no matter what's listed after
                // it. This was silently killing the real Jenkins container
                // every single time this stage ran. "stop" and "rm" DO
                // support per-service scoping, so we use those instead —
                // this achieves the same clean teardown without ever
                // touching anything outside database/frontend/backend.
                sh '''
                    docker compose -p foodgorilla stop database frontend backend || true
                    docker compose -p foodgorilla rm -f database frontend backend || true
                    docker compose -p foodgorilla pull database frontend backend || true
                    docker compose -p foodgorilla up -d --build database frontend backend
                '''
            }
        }
    


        // STAGE 4: RUN ANSIBLE PLAYBOOK (only on main, same reasoning as Stage 3)
        stage('Deploy Application via Ansible') {
            when {
                branch 'main'
            }
            steps {
                echo '🚀 Running post-deployment verification via Ansible...'
                // This does NOT deploy anything — Stage 3 already did that.
                // Same playbook file as the preflight stage, different tag.
                sh 'ansible-playbook -i "localhost," ansible/playbook.yml --tags smoke_test'
            }
        }
    }
    

    // 🌟 UNIFIED GLOBAL POST BLOCK (Merged GitHub notifications and console echoes)
    post {
        success {
            githubNotify(
                credentialsId: 'github-token',
                context: 'Jenkins CI/CD Pipeline',
                description: 'Build passed successfully!',
                status: 'SUCCESS',
                account: 'Benedictusernametaken',
                repo: 'Food-Gorilla',
                sha: env.GIT_COMMIT ?: sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
            )
            echo "🎉 Build #${BUILD_NUMBER} Passed! The 3-tier architecture is verified and secure."
        }
        failure {
            githubNotify(
                credentialsId: 'github-token',
                context: 'Jenkins CI/CD Pipeline',
                description: 'Pipeline Failed!',
                status: 'FAILURE',
                account: 'Benedictusernametaken',
                repo: 'Food-Gorilla',
                sha: env.GIT_COMMIT ?: sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
            )
            echo "❌ Build #${BUILD_NUMBER} Failed! Check the logs or integration test diagnostics above."
        }
    }
}