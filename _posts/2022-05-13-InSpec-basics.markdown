---
layout: post
title:  "Getting Started with InSpec"
date:   2022-05-15 20:20:20 +0530
categories: weekly update
---

## InSpec Baseline
- InSpec is an open-source framework for testing and auditing your applications and infrastructure.
- Turn Security and Compliance into code
  - Translate compliance into code
  - Clearly express statements of policy
  - Move risk to build/test from runtime
  - Find issues early
  - Write code quickly
  - Run code anywhere
  - Inspect machines, data & APIs


## InSpec Shell
- An interactive session that enables you to explore InSpec resources and controls on:
  - Your local workstation
  - A remote target 
  - A local docker container
  - A cloud provider

- To launch the inspec shell session locally: 
  ```inspec shell ```
- To launch the session on an instance:
  - ```inspec shell -t ssh://user:password@host```
  - ```inspec shell -t winrm://user:password@host```
  - ```inspec shell -t docker://container_id```
  - ```inspec shell -t cloud://region-subscription```

## InSpec Execution
- InSpec enables you to execute controls that you define
  - locally, in a control file or profile
    - ```inspec exec [control_file_name.rb]```
    - ```inspec exec [profile_directory_name]```
  - remotely, in a profile stored at a URL
    - ```inspec exec [URL]```
  - remotely, through the Chef Supermarket
    - ```inspec supermarket exec [profile_name]```
  - remotely, through Chef Compliance
    - ```inspec compliance exec [profile_name]```

- To see list of profiles available on the supermarket
  - ```inspec supermarket profiles```
- To see list of profiles available on the compliance
  - ```inspec compliance profiles```
- Some controls may require root privileges, in such situation add the sudo flag while executing the profile
  - ```inspec exec [profile] -t [target] --sudo```
- To execute multiple profiles:
  - ```inspec exec [profile_1_name] [profile_2_name]```

## Anatomy of Controls
InSpec expresses expectations about the state of the target system through resources expressed as controls.
- **resources** with defaults, required, optional parameters, and filters.
- **individual tests** with properties, conditions, matchers, and expected results.

### Resources
InSpec resources map platform concepts to code objects
- default value
- required parameters
- optional parameters
- resources with filters


### Controls
InSpec controls express expectations about the state of the resources on our system. 
- properties
- conditions
- matchers
- expected results

### Sample Control
```ruby
  control "tmp-1.0" do                        # A unique ID for this control
    impact 0.7                                # The criticality, if this control fails.
    title "Create /tmp directory"             # A human-readable title
    desc "An optional description..."
    describe file("/tmp") do                  # The actual test
      it { should be_directory }
    end
  end
```

### Components of Controls
Controls has 
- Properties like: ```baseurl, version, status```
- Conditions like ```should, should_not```
- Matchers like: ```exist, eq, be_installed, cmp```
- Expected results: the result we expect

## Create a Profile
InSpec Profiles organize multiple controls into a reusable artifact that can be described and versionsed

- To create a Profile:
  - ```inspec init profile [profile_name]```
  - exmaple: ```inspec init profile my_profile```
- To check if an instance exist
  - ```inspec detect -t [target_instance]```
  - example: ```inspec detect -t docker://[container_id]```
- To execute the profile on the target
  - ```inspec exec -t [target_instance] [profile_name]```
  - example: ```inspec exec -t docker://[container_id] my_profile```

## Profile Dependencies
InSpec profiles can bring in the controls from another InSpec profile. These dependencies are maintained in the metadata and captured in the profile's lockfile.

Steps:
- Let's consider we have two profiles viz. ```base_profile``` and ```child_profile``` which has different controls executed by them. 
- If you do not have two profiles, create using ```inspec init profile [profile_name]``` for learning purpose.
- We would like to execute ```child_profile``` which in turns should execute the controls present in the ```base_profile```.
- Add dependency in the metadata of the ```child_profile``` i.e. ```inspec.yml``` as below
  ```yml
  depends:
    - name: base_profile
      path: ../base_profile
  ```
- Edit the control file of the ```child_profile``` and add the following lines:
  ```ruby
  include_controls 'base_profile'
  ```
  This will include the controls present in the ```base_profile```
- Execute the ```child_profile``` which should execute the controls present in its own control file as well as the ```base_profile``` control file.
  ```
  inspec exec child_profile
  ```

Possible reasons of failure for the ```child_profile``` not being able to execute controls present in the ```base_profile```
- The ```inspec.lock``` file is not updated. To do this delete the file and execute the profile again. Other way to do this without deleting the ```inspec.lock``` file is to use ```--no-create-lockfile``` while executing the profile. Example:  
  ```
  inspec exec child_profile --no-create-lockfile
  ```
- The path to the profile is not set correctly in the ```inspec.yml``` file.

Other ways to add depedency of other profile are as follows: 
- via git:
  ```yaml
    depends:
    - name: base_profile
      git: https://gitrepo/profile
      branch: master
  ```
- via url:
  ```yaml
    depends:
    - name: base_profile
      url: [url]
  ```
- via supermarket:
  ```yaml
    depends:
    - name: base_profile
      supermarket: origin/profile_name
  ```
- via compliance:
  ```yml
    depends:
    - name: base_profile
      compliance: origin/profile_name
  ```

## Conditional Execution
InSpec controls can be conditionally executed based on additional requirements expressed through InSpec helpers and language constructs.

### Profile Attributes:
- only_if
  ```ruby
  only_if do
      command('git').exist?
  end
  ```
- if 
  ```ruby
  if os.linux?
    ...
  end  
  ```
- describe.one 
  ```ruby
  describe.one do
    describe file('primary.cfg) do
        its('content') { ... }
    end

    describe file('seconday.cfg) do
        its('content') { ... }
    end
  end
  ```
- supports
  ```yml
  supports:
    os-family: OSFAMILY
  ```

## My Activity for Profile Dependencies
For this activity I have two docker instances running, a CentOS and a Ubuntu image. The Ubuntu instance has ```git``` installed, and in the CentOS instance I have created a file ```hello.txt``` in the root directory with a content of "Conditional test" inside of it.

- Execution on CentOS
  ```
  inspec exec conditional_profile -t docker://86b8cd0c449f --no-create-lockfile
  ```
  Output:
  ```
  Profile: InSpec Profile (conditional_profile)
  Version: 0.1.0
  Target:  docker://86b8cd0c449fca335313a635461801e1a5c82cf58ec5a71e53ecbbda6a856381

    ✔  tmp-1.0: Testing Command on Conditional Execution with if
       ✔  Command: `cat /hello.txt` stdout is expected to match "Conditional Test"
    ✔  tmp-2.0: Testing Command on Conditional Execution with describe.one
       ✔  Command: `cat /hello.txt` stdout is expected to match "Conditional Test"
    ✔  tmp-3.0: Testing Command on Conditional Execution with only_if
       ✔  Command: `cat /hello.txt` stdout is expected to match "Conditional Test"

    File /tmp
       ✔  is expected to be directory

  Profile Summary: 3 successful controls, 0 control failures, 0 controls skipped
  Test Summary: 4 successful, 0 failures, 0 skipped
  ```

- Execution on Ubuntu:
  ```
  inspec exec conditional_profile -t docker://2d31e0883f76 --no-create-lockfile
  ```
  Output:
  ```
  Profile: InSpec Profile (conditional_profile)
  Version: 0.1.0
  Target:  docker://2d31e0883f76ba10891bc9ca5f8bad99ad1291b49fa6b84ff728e34bf189869c

    ✔  tmp-1.0: Testing Command on Conditional Execution with if
       ✔  Command: `git --version` stdout is expected to match "git version 2.25.1"
    ✔  tmp-2.0: Testing Command on Conditional Execution with describe.one
       ✔  Command: `git --version` stdout is expected to match "git version 2.25.1"
    ↺  tmp-3.0: Testing Command on Conditional Execution with only_if
       ↺  Skipped control due to only_if condition.

    File /tmp
       ✔  is expected to be directory

  Profile Summary: 2 successful controls, 0 control failures, 1 control skipped
  Test Summary: 3 successful, 0 failures, 1 skipped
  ```
- When I change supports os-family to windows in the ```inspec.yml``` file and execute on the linux instances i.e. Ubuntu & CentOS in my case. My inspec.yml file looked something like this:
  ```yml
  name: conditional_profile
  title: InSpec Profile
  maintainer: The Authors
  copyright: The Authors
  copyright_email: you@example.com
  license: Apache-2.0
  summary: An InSpec Compliance Profile
  version: 0.1.0
  supports:
    platform: os
    os-family: windows
  ```
  Execute:
  ```
  inspec exec conditional_profile -t docker://2d31e0883f76 --no-create-lockfile
  ```
  Output:
  ```
  Skipping profile: 'conditional_profile' on unsupported platform: 'ubuntu/20.04'.

  Test Summary: 0 successful, 0 failures, 0 skipped
  ```