See the (properly formatted) [documentation page](http://entagen.github.com/jenkins-build-per-branch/) for project details and usage. A simple text version is available below.

Jenkins Build Per Branch

The code in this repo lets you automatically generate Jenkins jobs for new branches in a specified repository, using templates to create the jobs.

Using it is as easy as:

- Creating the set of jobs for your master branch that will be used as templates for feature branches, test them so that they work with master
- Make a new job that is in charge of creating new builds for each branch in jenkins using the jenkins-build-per-branch code

This job will be in charge of both creating new jobs for new feature branches as well as cleaning up old feature branch jobs that have been fully merged into another branch

Installation

It has the following plugin requirements in Jenkins:

- Jenkins Git Plugin: https://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin
- Jenkins Gradle plugin: http://wiki.jenkins-ci.org/display/JENKINS/Gradle+Plugin
- the git command line app also must be installed (this is already a requirement of the Jenkins Git Plugin), and it must be configured with credentials (likely an SSH key) that has authorization rights to the git repository
- Additional optional functionality for grouping your feature-branch builds is enabled if you have the Nested View Plugin (https://wiki.jenkins-ci.org/display/JENKINS/Nested+View+Plugin) installed, but it is not required.

There are a few ways to use jenkins-build-per-branch but the easiest is probably to make a new Jenkins job that runs on a cron timer (every 5 minutes or so).

You'll want to already have your template jobs running and passing (likely one or more jobs on the master branch of your repo).

Then, create a new Jenkins job called SyncYOURPROJECTGitBranchesWithJenkins as a "free-style software project".

Set the job to clone the jenkins-build-per-branch repo from github with the repository url git@github.com:entagen/jenkins-build-per-branch.git.

The branch to build should be set to origin/master.

Check the box to have it "build periodically" and set the cron expression to */5 * * * * to get it building every 5 minutes.

Click on the "Add build step" dropdown and pick "Invoke Gradle script" from the dropdown.

In the "Switches" box, enter the system parameters unique to your jenkins setup and git install, here's an example that you can use to tweak:

-DjenkinsUrl=http://localhost:8080/jenkins -DgitUrl=git@github.com:mygithandle/myproject.git -DtemplateJobPrefix=MyProject -DtemplateBranchName=master -DnestedView=MyProject-branches -DdryRun=true
            
Those switches include a -DdryRun=true system parameter so that the initial run does not change anything, it only logs out what it would do.

In the "Tasks" field, enter syncWithRepo.

Save the job and then "Build Now" to see if you've got things configured correctly. Look at the output log to see what's happening. If everything runs without exceptions, and it looks like it's creating the jobs and views you want, remove the -DdryRun=true flag and let it run for real.

This job is potentially destructive as it will delete old feature branch jobs for feature branches that no longer exist. It's strongly recommended that you back up your jenkins jobs directory before running, just in case. Another good alternative would be to put your jobs directory under git version control. Ignore workspace and builds directories and just about everything can be added. Commit periodocally and if something bad happens, revert back to the last known good version.

Script System Parameter Details

The following options are available in the script:

-DjenkinsUrl - ex: http://localhost:8080/jenkins/ - the url to the jenkins repo, you should be able to append api/json to the url to get a json feed
-DjenkinsUser - ex: admin - the username for Jenkins HTTP basic auth - optional
-DjenkinsPassword - ex: sekr1t - the password for Jenkins HTTP basic auth - optional
-DgitUrl - ex: git@github.com:mycompany/myproject.git - the url to the git repo, read-only git url preferred
-DtemplateJobPrefix - ex: myproj - the prefix that the template jobs (and all newly created jobs) will have, likely the project name, the view containing all of the branch's jobs will also use this prefix.
-DtemplateBranchName - ex: master - the branch name with jobs in jenkins that's used as a template for all feature branches, this will be the suffix replaced for each branch
-DbranchNameRegex - ex: feature\/.+|release\/.+|master - only branches matching the regex will be accepted (your templateBranchName must pass this regex if given!) - optional
-DnestedView - ex: MyProject-feature-branches - optional - the name of the existing nested view that will hold a view per feature branch, reqires the Nested View plugin to be installed. This is useful to avoid an explosion of tabs in Jenkin's UI. Without this parameter, each branch will get it's jobs in it's own tab at the top.
-DnoViews - ex: true - if this flag is passed, it will not create a view for each branch
-DdryRun - ex: true - if this flag is passed with any value, it will not execute any changes only print out what would happen
-DstartOnCreate - ex: true - if this flag is passed with any value, newly created jobs (i.e. because a new feature branch was created) will be executed immediately instead of waiting for the next polling interval or git-hook trigger. Most users will want to set this.

Conventions

It is expected that there will be 1 or more "template" jobs that will be replicated for each new branch that is created. In most workflows, this will likely be the jobs for the master branch.

The job names are expected to be of the format:

<templateJobPrefix>-<jobName>-<templateBranchName>

Where:

- templateJobPrefix is probably the name of the git repository, ex: MyProject
- jobName describes what Jenkins is actually doing with the job, ex: runTests
- templateBranchName is the branch that's being used as a template for the feature branch jobs, ex: master

So you could have 3 jobs that are dependent on each other to run, ex:

- MyProject-RunTests-master
- MyProject-BuildWar-master
- MyProject-DeployApp-master

If you created a new feature branch (ex: newfeature) and pushed it to the main git repo, it would create these jobs in Jenkins:

- MyProject-RunTests-newfeature
- MyProject-BuildWar-newfeature
- MyProject-DeployApp-newfeature

It will also create a new view for the branch to contain all 3 of those jobs called "MyProject-newfeature". If you haven't used the nestedView parameter, it will be a new tab at the top of the main screen, otherwise it will be contained within that view.

Once newfeature was tested, accepted, and merged into master, the sync job would then delete those jobs and it's view on the next run.

What Kinds of Worflows Need a Build Per Branch?

This can be very useful if your team is using a workflow like Github Flow.

In this workflow:

- the master branch is always deployable
- New work is always done in a feature branch (i.e. ted/my_fancy_feature) and never committed directly to master
- When a feature is finished and ready, a pull request is made against master that is reviewed and tested by at least one other developer
- After a pull request is approved, the feature branch is merged to master and the feature branch is then deleted
- master should then be deployed as soon as possible

The advantages to this model are:

- all code is seen (and tested) by at least one other person
- Feature branches are very short-lived (mostly a day or two) and rarely get out of date making the majority of merges trivial
- new features are quickly available to end users, and can be quickly fixed if there is some sort of issue
One disadvantage is that with multiple, short-lived branches, manually creating CI to confirm that all tests continue to pass becomes far too much overhead. It must be automated for this model to work and be trusted. "Jenkins Build Per Branch" is the answer to that problem.

Potential Future Enhancements

- allow a git repo to hold your job definitions, it would then be a submodule of the repo under test. As the jenkins configuration changes to meet the needs of the repo under test, the job definition repo could be updated appropriately. This would tie a particular SHA in the job definition repo to the current branch so Jenkins always has the right job definitions for building.

- make this into a plugin with a new jenkins "job type" or "build step" that simplifies things down to filling in a set of form fields

- Support token-based authentication in addition to the basic auth support it already has