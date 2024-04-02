# Achieving Sustainability by Network Automation: Principles and Practice
=========================================================================

This project can be used as a demo or hands-on lab for how to use a
network orchestrator to continuously optimize a set of network services
with respect to combination of goals, some of which might be related to
sustainability.

In this demo/lab we are using Cisco NSO as the network orchestrator. On
top of NSO, we have an example service running. The example service is
implemented in three different ways, on network automation levels 3, 4
and 5. For more information about what is meant by network automation
levels, see the [Credits and references](#Credits-and-references)
section below.

When used as a demo, the demo operator can switch freely between the
different service implementation levels, to highlight the behavioral
differences, and if desired also show the small code modifications
required to move from one level to the next. Or, for a shorter demo, the
demo operator may opt to only demonstrate the final level, level 5.

When used as a lab, the lab participants would typically start on the
most basic implementation level, level 3, then read the high level
description of what needs to change in the enclosed lab guide, and
figure out the exact code changes required. The lab participants can
also look up the exact code changes required in the solutions section of
the lab guide. At any time, the lab participant may switch to a working
implementation of a different level, e.g. get a fresh start at level 4
or go directly to play with the final system on level 5.

# Skills Required and Gained

In order to act as the demo operator or lab participant, you will need
* Basic programming skills in Python
* Some familiarity with NSO services and YANG modules

We hope this demo/lab will give the operator, participants and audeince
some
* Hands-on experience with Network Automation Levels 3-5
* Giddy feeling of power arising from services that adapt to their
environment
* Insight into how network orchestration may be used for sustainability

# DISCLAIMER

This training document is to familiarize with Network Services
Orchestrator (NSO), Network Automation Levels and how they can be
combined to achieve a more sustainable network operation. Although the
lab design and configuration examples could be used as a reference, 
it’s not a real design, thus not all recommended features are used,
or enabled optimally. For the design related questions please contact
your representative at Cisco, or a Cisco partner.

# Background

Once the Adaptive Service Activation Scripts (Network Automation
Level 2) are left behind, a world of life-cycle flexibility and
stylistic freedom of the services running in our network. What can we
use this flexibility for?

In this lab, the participant will upgrade a pre-existing Network
Services Orchestrator (NSO) service from Network Automation Level 3 to
Level 5 by adding service health measurement and make it adapt to
changes in the environment.

The vision of the Internet Engineering Task Force (IETF) Application
Level Traffic Optimization (ALTO) Working Group (WG) is to optimize the
application and network together, so that the delivered service is
delivered as well as possible to the collection of users. This might
mean delivering this user's service from a data center that is not the
closest one, but from another with less load or lower energy prices, and
therefore less expensive to the customer. Moving service instances
around becomes a natural part of the life-cycle.

# Use Case

## Video Delivery Service

In this example, we are looking at a video delivery service. The service
is implemented by a network of data centers (DCs) and edge devices. Each
DC contains an Origin Video Server (this is something we made up for
this lab), and a generic firewall. The edge devices are Edge Video
Caches (again, this is something we made up for this lab), that are
built to work together with the Origin Video Servers. Imagined end-users
can then consume their video feeds through the Edge Video Caches.

```
    +-------------+                         +-------------+
    |             |       +----------+      |             |
    |   dc0       |       | skylight |      |   dc1       |
    |             |       +----------+      |             |
    | +---------+ |                         | +---------+ |
    | | origin0 | |                         | | origin1 | |
    | +----v----+ |                         | +----v----+ |
    |      |      |                         |      |      |
    |   +--v--+   |                         |   +--v--+   |
    |   | fw0 |   |                         |   | fw1 |   |
    |   +--v--+   |                         |   +--v--+   |
    +------|------+                         +-----/|\-----+
          /              +-----------------------+ | \
         /              /             +------------+  \
    +---v---+      +---v---+      +---v---+       +---v---+
    | edge0 |      | edge1 |      | edge2 |       | edge3 |
    +-------+      +-------+      +-------+       +-------+

    Fig 1. The network topology.
```

The NSO service we are talking about here controls how the Edge Video
Caches are connected to the DCs. Each Edge Video Cache must be connected
to exactly one DC, but which one can vary over time, depending on the DC
availability, delivered quality, load and the current energy price at
the DC location.

At the starting point of this lab, it is the NSO operators that manually
decide through configuration which DCs are considered available and
which Edge Video Caches are connected to which DCs. While the service at
this level automates the device configuration work for the Origin Video
Servers and firewalls, the manual decisions/configurations about DC
availability and which Edge device should connect to which DC obviously
is not automated to a high level. We call this level 3.

## Lab Starting-point

In the lab setup, we have a single NSO instance that controls all the
DCs, all the relevant devices in them, and all the Edge devices. All the
devices in the DC and the Edge devices are really NSO NETSIM devices.
That means they basically only implement the management interfaces as
described by their YANG models, but otherwise don’t really do anything.
We have endowed some of these NETSIMs with a little bit of additional
behavior, though, so that they react and respond to a few things. Such
functionality will be described later, as needed.

Also, logically part of each DC is an Accedian Skylight monitoring
system. In our setup, there is really only one instance of a skylight
system, or “device” as NSO tends to think of all managed systems as
devices. A single skylight is enough since we think of it as running “in
the cloud” outside our DCs.

When we start, the NSO system is already up and running with the level 3
service implementation, and there are Network Element Drivers (NEDs) for
all device types installed, and the devices are already listed in the
NSO device list.

# Installation

## Step P.1 Installing development tools

In order to perform the build steps required in this demo/lab, you will
need a few suitable development tools. Either MacOS or Linux is fine.

We recommend using VSCode for working with the code, showing it and to
diff the different implementation levels with each other, but VSCode is
by no means required.

make, the tool that runs executes Makefiles is required. No
fancy features are used, so pretty much any brand and version of make
should do.

Python 3.7 or higher is required since most of the application code
consists of Python applications running on top of NSO.

xsltproc is required for building some of the packages. Again no fancy
features are used, so pretty much any version should be fine.

## Step P.2 Installing NSO

If you already have an NSO environment up and running, you can skip this
section. The development team used NSO 6.1.5, but other similar versions
are likely to work just as well.

If don't have a suitable version of NSO 6.x installed, you need to
download and install NSO on your MacOS or Linux system. You will want to
follow the instructions for a [local install](https://developer.cisco.com/docs/nso/guides/#!installation/local-install-steps).

Since we will be developing an application on top of NSO, the section
called Additional Requirements on the page above applies, but for the
specific example we are working with here, only the xsltproc and Python
related requirements apply.

You do not need to install any Network Element Drivers (NEDs). All the
NEDs needed for this example are included in the build environment.

You do not need to create any runtime directory or launch any NSO
instance. The git repository cloned in the next section will be your
runtime directory, and the Makefile will launch the NSO instance and
perform other setup.

## Step P.3 Cloning the Example Application Repository

The git repository that hosts this demo/lab contains an NSO runtime
directory and a PDF document called NSO Sustainability Automation
Example Lab Guide. Clone the git repository, and follow the instructions
in the Lab Guide.

$ git clone git@github.com:janlindblad/nso-sustainability-automation-example.git

# Usage

This example can be used like a lab or a demo. The following subsections
describe how you would use the system for either purpose. In both cases,
you would start by opening up the Lab guide document "NSO Sustainability
Automation Example Lab Guide v?.pdf". The chapter called "Preparing your
Runtime Environment" explains how to build and start the example system.

## The Lab Track

This lab consists of two tasks. The first takes the level 3 service
implementation to level 4. The second task takes it further to level 5.

The chapter called "Task 1 Overview: Go to Level 4, Add Monitoring"
explains the first lab task. If you think the lab is a bit too
challenging, or if you would simply like to compare notes with the lab
authors, you can peek into section "Task 1 Detailed Instructions", where
each step required to do the lab is explained in full details. 

The next chapter, "Task 2 Overview: Go to Level 5, Add Ongoing
Optimization", is structured in a similar way.

At any point, you can switch back and forth between the implementation
levels. This can be useful if you get stuck or would like to compare
your implementation with the one provided by the lab authors.

## The Demo Track

For a really quick demo, you could switch to level 5 implementation and
show how the system behaves by following the instructions in the
"Overview 2.7 Test" section. That should give a good idea about what
sustainable network automation could look like.

For a slightly longer demo, you could start with a demo of the level 4
implementation based on the "Overview 1.7 Test" section, followed by the
level 5 demo described above.

If the audience is interested in understanding how to build a level 4 or
level 5 system, the Task 1 and Task 2 "Detailed Instructions" sections
provide the full details of what has been added (and removed) between
the implementation levels, but also some discussions about why, which
could be interesting to bring up.

For the deeply interested NSO practitioners, it may be relevant to also
show all the file level changes side by side (VSCode is good at this).
The changes are pretty small (some would say shockingly small), which
should be fairly visible in such a side by side comparison.

# Credits and references

Major code contributors:
+ Per Andersson
+ Viktoria Fordos
+ Jan Lindblad
+ Ulrik Stridsman

This lab is based on a [blog post on Network Automation Levels by
Kristan Larsson and Jan Lindblad](https://community.cisco.com/t5/crosswork-automation-hub-blogs/network-automation-levels/ba-p/4742665),
which was later reworked into a [Cisco Live presentation](https://www.ciscolive.com/on-demand/on-demand-library.html?search=2431#/session/1675722382991001th6H). 
Many thanks to Marisol Palmero Amador for curating this work.