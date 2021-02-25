node {

    def built_image
    def image = 'jugendpresse/openslides'

    def gitrepository = 'https://github.com/OpenSlides/OpenSlides.git'
    def gitbranch     = 'master'
    def begin_commit  = 'a37e219'

    def built_tags = []
    def build_tags = []
    def scm_repo   = ''
    def scm_push   = false
    def scm_branch = ''

    def contexts = []

    def versions = [
        'server',
        'client'
    ]

    stage('prepare run') {
        /* Clone current Jenkinsfile repository */
        def scmVar = checkout scm
        /* build variables for further work in clean up stage */
        scm_repo   = scmVar.GIT_URL
        scm_repo   = scm_repo.replaceAll(~/https:\/\//, "")
        scm_branch = scmVar.GIT_BRANCH
        int ind = scm_branch.lastIndexOf("/")
        if ( ind >= 0 ) {
            scm_branch = new StringBuilder(scm_branch).replace(0, ind+1, "").toString()
        }
        built_tags = readJSON file: 'built_tags.json'
        /* Clone current project */
        sh 'rm -rf app && git clone ' + gitrepository + ' --branch ' + gitbranch + ' --single-branch app/'
        build_tags = sh(
                script: 'cd app && git fetch --tags && git tag --contains ' + begin_commit + ' && cd ..',
                returnStdout: true
            ).split('\n')
    }

    stage('read YAML information') {
        def cur_path = 'app/docker/'
        def data     = readYaml file: cur_path + 'docker-compose.dev.yml'
        contexts     = [
            "client": [
                "context":    sh(
                                 script: 'realpath --relative-to=. ' + cur_path + data.services.client.build.context,
                                 returnStdout: true
                              ),
                "dockerfile": data.services.client.build.dockerfile
            ],
            "server": [
                "context":    sh(
                                 script: 'realpath --relative-to=. ' + cur_path + data.services.server.build.context,
                                 returnStdout: true
                              ),
                "dockerfile": data.services.server.build.dockerfile
            ],
        ]

        echo "Server Context \"" + contexts.server.context + "\" and Dockerfile \"" + contexts.server.dockerfile + "\""
        echo "Client Context \"" + contexts.client.context + "\" and Dockerfile \"" + contexts.client.dockerfile + "\""
    }

    stage('build docker images for tags not already built') {

        def vtag
        int ind

        for ( int i = 0; i < build_tags.size(); i++ ) {

            vtag = ''

            if (built_tags[build_tags[i]]) {
                // Already exists – do nothing at the moment
                echo "Tag " + build_tags[i] + " already built on " + built_tags[ build_tags[i] ]
            }
            else {

                echo "Tag " + build_tags[i] + " to be built now."
                scm_push = true
                sh 'cd app && git checkout ' + build_tags[i] + ' && cd ..'

                for ( int j = 0; j < versions.size(); j++ ) {
                    def version      = versions[j]
                    def image_string = image + ':' + version + '_' + build_tags[i]
                    echo 'Building Image "' + image_string + '"'

                    dir( contexts[ version ][ 'context' ].trim() ) {

                        sh 'ls -lah'
                        sh 'pwd'

                        def df = contexts[ version ][ 'dockerfile' ]

                        built_image = docker.build( image_string, "-f - . < ${df}" )
                        withCredentials([usernamePassword( credentialsId: 'jpdtechnicaluser', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                            docker.withRegistry('', 'jpdtechnicaluser') {
                                sh "docker login -u ${USERNAME} -p ${PASSWORD}"
                                built_image.push()
                                if (build_tags[i] =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/) {
                                    ind = build_tags[i].lastIndexOf(".")
                                    vtag = build_tags[i]
                                    while ( ind > 0 ) {
                                        vtag = new StringBuilder(vtag).substring(0, ind).toString()
                                        built_image.push(vtag)
                                        ind = vtag.lastIndexOf(".")
                                    }
                                    built_image.push(version)
                                }
                            }
                        }

                        built_image = null
                    }
                }

                def now = new Date()
                built_tags[build_tags[i]] = now.format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Europe/Berlin'))
            }
        }
    }

    stage('clean up') {
        sh 'rm -rf app/'
        if (scm_push) {
            writeJSON file: 'built_tags.json', json: built_tags, pretty: 4
            withCredentials([usernamePassword(credentialsId: 'jpdtechnicaluser', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                sh(
                    'git config user.name Jenkins && ' +
                    'git config user.email server@jugendpresse.de && ' +
                    'git add built_tags.json && ' +
                    'git commit -m "Jenkins: automated build from ' + built_tags[build_tags[ build_tags.length - 1 ]] + '" && ' +
                    'git push https://${GIT_USERNAME}:${GIT_PASSWORD}@' + scm_repo + ' HEAD:' + scm_branch
                )
            }
        }
    }
}
