@Library('github.com/coreos/coreos-ci-lib') _

repo = "coreos/fedora-coreos-config"
branches = [
    "testing-devel",
    "next-devel"
]
botCreds = "github-coreosbot-token"

properties([
    pipelineTriggers([
        // we don't need to bump lockfiles any more often than daily
        cron("H H * * *")
    ])
])

try { timeout(time: 120, unit: 'MINUTES') { cosaPod {
    parallel branches.collectEntries { branch -> [branch, {
        shwrap("mkdir ${branch}")
        dir(branch) {
            shwrap("cosa init --branch ${branch} https://github.com/${repo}")

            shwrap("""
              git -C src/config config --global user.name "CoreOS Bot"
              git -C src/config config --global user.email "coreosbot@fedoraproject.org"
            """)

            // do a first fetch where we only fetch metadata; no point in
            // importing RPMs if nothing actually changed
            stage("Fetch Metadata") {
                shwrap("cosa fetch --update-lockfile --dry-run")
            }

            if (shwrapRc("git -C src/config diff --exit-code") == 0) {
                println("No changes")
                return
            }

            // sanity-check only base lockfiles were changed
            shwrap("""
              # do this separately so set -e kicks in if it fails
              files=\$(git -C src/config ls-files --modified --deleted)
              for f in \${files}; do
                if ! [[ \${f} =~ ^manifest-lock\\.[0-9a-z_]+\\.json ]]; then
                  echo "Unexpected modified file \${f}"
                  exit 1
                fi
              done
            """)

            stage("Fetch") {
                // XXX: hack around subtle lockfile bug (jlebon to submit an
                // rpm-ostree issue or patch about this)
                shwrap("cp src/config/fedora-coreos-pool.repo{,.bak}")
                shwrap("echo cost=500 >> src/config/fedora-coreos-pool.repo")
                shwrap("cosa fetch --strict")
                shwrap("cp src/config/fedora-coreos-pool.repo{.bak,}")
            }

            stage("Build") {
                shwrap("cosa build --strict")
            }

            fcosKola(cosaDir: ".")

            stage("Build Metal") {
                shwrap("cosa buildextend-metal")
                shwrap("cosa buildextend-metal4k")
            }

            stage("Build Live") {
                shwrap("cosa buildextend-live --fast")
                // Test metal4k with an uncompressed image and metal with a
                // compressed one
                shwrap("cosa compress --artifact=metal")
            }

            try {
                parallel metal: {
                    shwrap("kola testiso -S --scenarios pxe-install,iso-install,iso-offline-install --output-dir tmp/kola-testiso-metal")
                }, metal4k: {
                    shwrap("kola testiso -S --scenarios iso-install,iso-offline-install --qemu-native-4k --output-dir tmp/kola-testiso-metal4k")
                }
            } finally {
                shwrap("tar -cf - tmp/kola-testiso-metal/ | xz -c9 > ${env.WORKSPACE}/kola-testiso-metal.tar.xz")
                shwrap("tar -cf - tmp/kola-testiso-metal4k/ | xz -c9 > ${env.WORKSPACE}/kola-testiso-metal4k.tar.xz")
                archiveArtifacts allowEmptyArchive: true, artifacts: 'kola-testiso*.tar.xz'
            }

            // OK, it passed kola: just push to the branch. In the future, we might be
            // fancier here; e.g. if tests fail, just open a PR, or if tests passed but a
            // package was added or removed.
            stage("Push") {
                shwrap("git -C src/config commit -am 'lockfiles: bump to latest'")
                withCredentials([usernamePassword(credentialsId: botCreds,
                                                  usernameVariable: 'GHUSER',
                                                  passwordVariable: 'GHTOKEN')]) {
                  // should gracefully handle race conditions here
                  sh("git -C src/config push https://\${GHUSER}:\${GHTOKEN}@github.com/${repo} ${branch}")
                }
            }
        }
    }] }
    currentBuild.result = 'SUCCESS'
}}} catch (e) {
    currentBuild.result = 'FAILURE'
    throw e
} finally {
    if (currentBuild.result != 'SUCCESS') {
        slackSend(color: 'danger', message: ":fcos: :trashfire: <${env.BUILD_URL}|bump-lockfile #${env.BUILD_NUMBER}>")
    }
}
