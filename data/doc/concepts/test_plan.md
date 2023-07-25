# Test Plan  <!-- omit in toc -->

- [INFORMATION](#information)
    - [PROJECT DESCRIPTION](#project-description)
- [UNIT TEST](#unit-test)
    - [UNIT TEST STRATEGY](#unit-test-strategy)
- [INTEGRATION TEST](#integration-test)
  - [INTEGRATION TEST STRATEGY](#integration-test-strategy)
- [REGRESSION TEST](#regression-test)
  - [REGRESSION TEST STRATEGY](#regression-test-strategy)
  - [REGRESSION TEST CASES](#regression-test-cases-controller-tests)
- [Appendix](#appendix)

## INFORMATION 

### PROJECT DESCRIPTION

The updateagent is an edge device service written in Go. 
It enables retrieval and installation of remote software updates for controllers of the KUKA RoX plattform.

### SCOPE

Automated functional Tests:
- Unit Testing
- Integration Testing
- Regression Testing(Controller Tests)

Manual Tests: 
- Controller Tests

### OUT OF SCOPE

- API Testing
- End-to-End Testing 
- Performance Testing
- Security Testing

### ENVIRONMENT AND TOOLS

- Azure Devops for User Stories and Defect Management [see Bug Flow in Polarion](https://polarion.kuka.int.kuka.com/polarion/#/project/p0649.0_kuka_rox/wiki/Project%20Management/bug_handling_roX?sidebar=signatures)
- Vagrantbox with debian/bullseye64
- Tests are written in Golang 
- [Stretchr/testify](https://github.com/stretchr/testify) GO-Package for Assertions and Mocking
- As Testdata we use Bundles and KUKA-OS provided by KUKA and our own synthetic bundles and KUKA-OS-Dummy, <br>
  certificates, Images and other files stored in our [update-agent Repository](https://dev.azure.com/kuka/RoX%20OS/_git/operation_management_update_agent)
- Controller Tests on real device KRC5-Hardware from KUKA in our MaibornWolff Office in Munich

## UNIT TEST

### UNIT TEST STRATEGY 

The code is covered by unit tests for our packages, which reside in `*_test.go` files. 

## INTEGRATION TEST

Combines individual software modules and test as a group.

### INTEGRATION TEST STRATEGY

Evaluates all integrations with locally developed shared libraries, with consumed services, and other touch points.
- Stored in a separate folder and package itests
- Tests cover our Business API 
- Runs on a non-controller OS in Vagrantbox
- Mocking interfaces are used to interact with system software such as podman and Systemd.
- The tests should be performant
- When a new feature is implemented there must be an integration test

#### Constrains 

- Instead of real KUKA bundles and images the integration-test uses synthetic bundles as test data
- Not the real KUKA-OS and swu files are used in integration-tests
- The USB case is currently covered almost exclusively.
- Incompatible for CI process and triggered manually

## REGRESSION TEST

Ensure that previously developed and tested software still performs after change.

### REGRESSION TEST STRATEGY

- Regression tests are carried out with automated tests on the KUKA hardware KRC5 before each KUKA-OS release.
- The functionalities in terms of installability, upgradability and compatibility are tested. 
- Tests are done with the current released KUKA Robotic-Base-Bundle, Addon-Bundles and KUKA-Linux.
- The listed test cases below cover the USB and remote case.
- Test cases have to be checked for updates when a new release is planned

### REGRESSION TEST CASES (CONTROLLER TESTS)

| \#  | NAME                                               | GIVEN/WHEN/THEN                                                                                                                                                                                                                                                                                         | TEST DELIVERABLES                                                                                                                     |
|-----|----------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Only Bundles (application software)                | GIVEN: <br/> - robotics-base-bundle on USB/remote location available <br> WHEN: <br/> installation is started by update-agent <br> THEN: <br/>robotic-base-bundle status is installed and all services are up and running                                                                               | release candidate KUKA-Linux and robotics-base-bundle                                                                                 |
| 2   | Install Toolbox/Addon Bundles                      | GIVEN: <br/> - robotics-base-bundle installed <br/> - addon bundles on USB/remote location available <br> WHEN: <br/> installation is started by update-agent <br> THEN:<br/> addon-bundle status is installed and all services are up and running                                                      | release candidate KUKA-Linux, robotic-base-bundle and latest addon bundles                                                            |
| 3   | Install KUKA-Linux                              | GIVEN: <br/> latest KUKA-Linux installed on controller <br> WHEN: <br/> installation of KUKA-Linux is started by update-agent  <br> THEN: <br/> KUKA-Linux is installed <br/>                                                                                                                           | - latest KUKA-Linux and release candidate KUKA-Linux <br/>  |
| 4   | Combined Update of KUKA-Linux and Bundles          | GIVEN: <br/> latest KUKA-Linux with compatible bundles installed on controller <br> WHEN: <br/> installation of KUKA-Linux and robotics-base-bundle is started by update-agent <br> THEN: <br/> KUKA-Linux is installed and robotic-base-bundle status is installed and all services are up and running | - latest KUKA-Linux and release candidate KUKA-Linux <br/> - stable robotic-base-bundle, addon-bundle and latest  robotic-base-bundle |
| 5   | Combined Update of KUKA-Linux and Bundles (failed) | GIVEN: <br/> latest KUKA-Linux with compatible bundles installed on controller <br> WHEN: <br/> installation of KUKA-Linux and robotics-base-bundle fails with error <br> THEN: <br/> a rollback is performed and the previous installed KUKA Linux and bundles are shown as installed                  | latest KUKA-Linux, release candidate KUKA-Linux  <br/> stable robotic-base-bundle, addon-bundle and latest <br> robotic-base-bundle   |


## Risks

- The system test (end-to-end) from KUKA is very extensive and not automated.
- When the system test is performed there could be not enough time to fix bugs that occur during tests.
- There is no special focus on the update-agent application in the KUKA system test. 
- There is no defined process yet how the test data is provided.  

## Outlook

- We plan to use the KUKA test infrastructure and KRC5 controller(office variant) in Augsburg.
- Our tests should be accessible for KUKA 


## Appendix

### Technical Implementation

#### Current implementation regression test

* Test procedure
  * Establish a connection with Controller
    * SSH portforward
  * Prepare Controller
    * Factory Reset with iiqka Tools and install KUKA OS via WebUI `http://<controller_ip>:8080`
  * Prepare Tests
    * Bundle-Files(synthetic) in Vagrantbox created
    * Robotic-base-bundle with Ansible
    * Images (tar.gz)
    * Files und Images are copied to the Controller(USB-directory)
  * Test execution
    * Go via GRPC-Interface
  * TestCleanup
    * Bundles are uninstalled in the test
    * Bundles are not deleted, when a new version of the bundle exists it will be overwritten

* Artifacts
  * Pipeline
  * Agent-Configuration (=Orchestrator, node.JS Runner executes Tasks in pipeline )
  * VM/Docker (self-hosted Agent running in a Dockercontainer)
  * Preparation/Cleanup Scripts
  * Go-Tests
  * TestImages
    * FileProvider (image contains Files)
    * Container (e.g. Check contents of Directory)
  * TestBundles
  * TestImage Build
  
#### Next Steps

 * Create a new pipeline for update-agent testing which prepares the controller
 * Write Tests in Go and execute them on a controller from KUKA test farm