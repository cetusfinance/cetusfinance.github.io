---
title: The details of setting up a CI .Net Project with CodeCoverage
excerpt: " Digging in"
category: "tim"
header:
  overlay_image: diggingIn.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: diggingInTeaser.jpg
tags: [setup, CI, testing, oss, .net]
---

Enough waffle so lets start. First we create a repo in Github, which was the easy bit (the hard bit was the 2 days of trying to think of a name)

![Creating the repo](https://cetus.io/images/diggingIn/CreateRepo.jpg)

So we created the basic repo, public of course and named it Qwack. I could make up some great creation myth and
maybe in the future I will but for now just know it came an interesting discussion that ended up and ducks and 
popped out of there.

Next I cloned the solution locally, for now I will be working directly on master but will change this to branches 
and pull requests with code reviews once we get into the meat of the code. So now that I had it locally I setup a 
very simple Visual studio solution and folder structure that looked something like this.

![The basic solution layout](https://cetus.io/images/diggingIn/basicsolution.png)

Now we had a structure that had places for our tests, our core source code and some samples. We also had some basic 
solution items which I will go into detail shortly. You can see that I added two .Net Core class libraries to get started. 
One for the tests for our math functions and another that is for our Interpolators which will give you a clue to where 
we will start!

The Next step was to decide on a framework version that we wanted to support and by that a matching .net standard 
version as well. Because we want to heavily use features such as interpolators the minimum version we decided that was 
useful for us was .Net 4.61.
This means that we will try to target 4.61 CLR and .Net Standard 1.4. You can see below the choices available to us

![.net Standard](https://cetus.io/images/diggingIn/netstandard.png)

We needed to setup our project.json file in our test to have all of the dependencies we would need later

``` Json
{
    "version": "0.1.0-*",
    "testRunner": "xunit",
    "dependencies": {
       "NETStandard.Library": "1.6.0",
       "coveralls.io": "1.3.4",
       "OpenCover": "4.6.519",
       "Microsoft.CodeCoverage": "1.0.2",
       "xunit": "2.2.0-beta2-build3300",
       "dotnet-test-xunit": "2.2.0-preview2-build1029",
       "Qwack.Math.Interpolation": {"target" : "project"}
  },
  "frameworks": {
  "netcoreapp1.0": {
    "dependencies": {
      "Microsoft.NETCore.App": {
        "version": "1.0.0",
        "type": "platform"
        }
      }
    }
  }
}
```

The testRunner tag does what it says on the tin and tells dotnet to use xunit for our testing. We have included the 
xunit nuget packages to support this also. Next we have the OpenCover nuget packages to support our code coverage 
statistics that we will automate so that they are then uploaded to coveralls.io which will track our code coverage 
history and can provide a simple "quality gate" for releases. I would like to integrate sonarcube in the future for
more detailed code quality tracking but at the moment it is difficult to integrate it with the project.json files. 
As MS has announced the switch back to msbuild for projects we will wait until this takes place and try to integrate it 
at that point.

So next we need to enable a bunch of integrations, and this is where modern day development, community, and the cloud 
really shine. It means your OSS project can have a true environment that outshines most corporate development environments 
around.

![our echo system](https://cetus.io/images/diggingIn/ecosystem.png)

Our Ecosystem

1. AppVeyor for building and running our tests on windows, as well as building our nuget and code coverage which we will see shortly
2. Coveralls for uploading our code coverage reports to and keeping a history of those
3. Gitter to provide a chat room where hopefully you and others will join to talk about ideas/improvements we can make
4. MyGet to upload our CI build nuget packages to
5. Travis CI which will provide our OSX and Ubuntu builds and tests to make sure we stay Xplat all the way
 
The best thing about all of this, for our open source project it's free! I can see why they do this other than the warm 
fuzzies of helping opensource. I would definitely recommend all/any of these sites to any future client that wants to 
get serious about builds and code quality and doesn't have a massive internal department that supplies all these services.

My configuration for travis and for AppVeyor can be seen over at the repo,

[.travis.yml](https://github.com/cetusfinance/qwack/blob/master/.travis.yml) for the travis file.
It basically lays out that we want a Ubuntu trusty image and an OSX xcode7.2 image.
Other than that we install .net core on the image and tell the dotnet function to update our nuget 
packages and run our tests. All pretty simple stuff.

[appveyor.yml](https://github.com/cetusfinance/qwack/blob/master/appveyor.yml) is the AppVeyor file and is a little more 
interesting. We follow a very similar pattern to the travis file except here we have two extra steps. The first is that we 
run OpenCover using

```
- ps: iex ((Get-ChildItem ($env:USERPROFILE + '\.nuget\packages\OpenCover'))[0].FullName + '\tools\OpenCover.Console.exe' + ' -register:user -target:"dotnet.exe" -searchdirs:".\test\Qwack.Math.Tests\bin\Debug\netcoreapp1.0" -oldstyle -targetargs:"test test/Qwack.Math.Tests" -output:coverage.xml -skipautoprops -returntargetcode -filter:"+[Qwack*]* -[*Tests]*"')
```

I have to give credit to this blog post [stephencleary](http://blog.stephencleary.com/2015/03/continuous-integration-code-coverage-open-source-net-coreclr-projects.html) 
for giving me the idea of using the powershell above to ensure that if I change the package version I won't need to 
go into the AppVeyor file and change the paths. This is useful in any CI build process, there is nothing worse than have 
to be updating your CI build all the time because it is brittle to tiny changes like that.

The settings are pretty standard I am ignoring auto properties because there is no point testing those, and filtering out 
any library that is not part of our namespace, and all those that end with test which is the format that we shall use 
going forward.

The next step is to upload the results to coveralls.io to do this we use the same powershell trick to get the package 
version

```
- ps: iex ((Get-ChildItem ($env:USERPROFILE + '\.nuget\packages\coveralls.io'))[0].FullName + '\tools\coveralls.net.exe' + ' --opencover coverage.xml')
```

We also set an environment variable at the top of the file for our token like so

```
environment:
  COVERALLS_REPO_TOKEN:
    secure: 9Flz1qTscHObl14SzMvP5PvOHXDf95W83hg6tUn1fg4Uovtu2tcSTexjpnlU/u/5
``` 

As putting my actual token in a public file on github would be a bad idea I simply go to AppVeyor and go to the encrypt 
option under my username

![encrypt in appveyor](https://cetus.io/images/diggingIn/encrypt.png)

then you enter your token and it gives you a new encrypted string. This then is entered as above but under a "secure" tag. 
AppVeyor will decrypt that during the build and inject it into the environment variables. You do need to be careful here 
as it can show up in the logs!

The last step that is not in the other two builds is to produce the nuget packages for the projects and upload them to myget. 
We use the same encrypted token concept for our API token here. We will upload to nuget as well but we will wait until we 
have stable releases and something worth releasing before we move onto that.

So finally we have all of this up and running I quickly added a failing test and went about adding badges for it, 
because without badges what sort of open-source project are you really?

``` markdown
[![Build Status](https://travis-ci.org/cetusfinance/qwack.svg?branch=master)](https://travis-ci.org/cetusfinance/qwack)
[![Build status](https://ci.appveyor.com/api/projects/status/dkh48o3mel1bkvv0/branch/master?svg=true)](https://ci.appveyor.com/project/Drawaes/qwack/branch/master)
[![Coverage Status](https://coveralls.io/repos/github/cetusfinance/qwack/badge.svg?branch=master)](https://coveralls.io/github/cetusfinance/qwack?branch=master)
[![Gitter chat](https://badges.gitter.im/cetusfinance/qwack/repo.png)](https://gitter.im/cetusfinance/qwack)
# qwack pronouced /kju:.wak/
A modern quantitative finance framework that makes the complex simple
```

![readme file with badges](https://cetus.io/images/diggingIn/readme.png)

so now our readme.md file has a bunch of badges for now and as you can see failing builds. So I faked the test 
passing and now I sleep knowing our project is ready for some actual code in it.

Head over to 

[Qwack Repo](https://github.com/cetusfinance/qwack) if you want to see the current state of the builds and coverage. 
Feel free while you are there to add things you would like to see to the issues page, or join us on Gitter!

The next article you see will be just coding we promise...!


