#!/usr/bin/env groovy

pipeline
{
    agent none

    options
    {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
    }

    stages
    {
        stage('Creating nodes')
        {
            agent { label "master" }
            steps
            {
                script
                {
                    JOB_ID = "${env.BUILD_TAG}"
                    jenkinsLib = load("/home/jenkins/jenkins_426.groovy")

                    jenkinsLib.CreateUbuntuBuildNode(JOB_ID)
                    jenkinsLib.CreateWindowsBuildNode(JOB_ID)
                }
            }
        }
        stage('Building CARLA')
        {
            parallel
            {
                stage('windows')
                {
                    agent { label "windows && build && ${JOB_ID}" }
                    environment
                    {
                        UE4_ROOT = 'C:\\UE_4.26'
                    }
                    stages
                    {
                        stage('windows setup')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    git update-index --skip-worktree Unreal/CarlaUE4/CarlaUE4.uproject
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make setup
                                """
                            }
                        }
                        stage('windows build')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    make LibCarla
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make PythonAPI
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make CarlaUE4Editor
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make plugins
                                """
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'PythonAPI/carla/dist/*.egg'
                                    stash includes: 'PythonAPI/carla/dist/*.egg', name: 'windows_eggs'
                                }
                            }
                        }
                        stage('windows retrieve content')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    call Update.bat
                                """
                            }
                        }
                        stage('windows package')
                        {
                            steps
                            {
                                bat """
                                    call ../setEnv64.bat
                                    make package
                                """
                                bat """
                                    call ../setEnv64.bat
                                    make package ARGS="--packages=AdditionalMaps,Town06_Opt,Town07_Opt,Town10HD_Opt --target-archive=AdditionalMaps --clean-intermediate"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts 'Build/UE4Carla/*.zip'
                                }
                            }
                        }
                        stage('windows deploy')
                        {
                            when { anyOf { branch "master"; branch "dev"; buildingTag() } }
                            steps {
                                bat """
                                    call ../setEnv64.bat
                                    git checkout .
                                    make deploy ARGS="--replace-latest"
                                """
                            }
                        }
                    }
                    post
                    {
                        always
                        {
                            deleteDir()

                            node('master')
                            {
                                script
                                {
                                    JOB_ID = "${env.BUILD_TAG}"
                                    jenkinsLib = load("/home/jenkins/jenkins_426.groovy")

                                    jenkinsLib.DeleteWindowsBuildNode(JOB_ID)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
