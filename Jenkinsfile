#!groovy

import groovy.json.JsonOutput
import groovy.json.JsonSlurper

stage 'Prepare build'
node {
    step([$class: 'GitHubSetCommitStatusBuilder'])

    deleteDir()
    checkout scm
    stash "project_files"
}

stage 'Acceptance Tests'
userInput = input(message: 'Launch acceptance tests?', parameters: [
    [
        $class: 'ChoiceParameterDefinition',
        name: 'storage',
        choices: 'odm\norm',
        description: 'Storage used for the build, MongoDB (default) or MySQL'
    ],
    [
        $class: 'TextParameterDefinition',
        name: 'features',
        defaultValue: 'features',
        description: 'Behat scenarios to build'
    ],
    [
        $class: 'ChoiceParameterDefinition',
        name: 'attempts',
        choices: '3\n1\n2\n4\n5'
    ],
    [
        $class: 'ChoiceParameterDefinition',
        name: 'php_version',
        choices: '5.6\n7.0',
        description: 'PHP version to run the tests with'
    ],
    [
        $class: 'ChoiceParameterDefinition',
        name: 'mysql_version',
        choices: '5.5\n5.7',
        description: 'MySQL version to run the tests with'
    ]
])

node {
    unstash "project_files"

    phpBinary = 'php'
    if ('5.6' != userInput['php_version']) {
        phpBinary += userInput['php_version']

        composerFile = readFile('composer.json')
        composerFile = updateComposerFile(composerFile)

        writeFile file: 'composer.json', text: composerFile
    } else {
        phpBinary += '5'
    }

    sh "${phpBinary} /usr/local/bin/composer update -o -n --no-progress --prefer-dist --ignore-platform-reqs"

    sh "/usr/bin/php7.0 /var/lib/distributed-ci/dci-master/bin/build" +
        " -p " + userInput['php_version'] +
        " -m " + userInput['mysql_version'] +
        " " + env.WORKSPACE +
        " " + env.BUILD_NUMBER +
        " pim-community-dev" +
        " " + userInput['storage'] +
        " " + userInput['features'] +
        " pim-community-dev/job/" + env.JOB_BASE_NAME +
        " " + userInput['attempts']
}

stage 'Results'
node {
    step([$class: 'ArtifactArchiver', allowEmptyArchive: true, artifacts: 'app/build/screenshots/*.png,app/build/logs/consumer/*.log', defaultExcludes: false, excludes: null])
    step([$class: 'JUnitResultArchiver', testResults: 'app/build/logs/behat/*.xml'])
    step([$class: 'GitHubCommitStatusSetter', resultOnFailure: 'FAILURE', statusMessage: [content: 'Build finished']])
}

/**
 * Updates the composer.json file according to user settings.
 *
 * @param String composerFile
 *
 * @return String
 */
def static updateComposerFile(composerFile)
{
    JsonSlurper jsonSlurper = new JsonSlurper();
    def parsedComposerFile = jsonSlurper.parseText(composerFile);

    parsedComposerFile['require-dev']['alcaeus/mongo-php-adapter'] = '1.0.*';

    return JsonOutput.prettyPrint(JsonOutput.toJson(parsedComposerFile));
}
