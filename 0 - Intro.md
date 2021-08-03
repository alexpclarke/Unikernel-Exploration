# 0 - Intro 

## 0.1 - Isolation

When working on an embeded system that features multiple components of varying safety criticallity, we can drastically improve the safety of the critical components by isolating them. For example, in a modern vehicle, the control of components such as the engine, steering, and brakes are controlled by the CAN bus. It is also common to find a media player that is able to connect to the internet in order to fetch media, directions and system updates. If we were to put the CAN bus in the same system as an unsafe media player, an attacker could use the media player as a way to access the CAN bus and crash the vehicle. On the other hand, if we isolate these two components, hacking the media player will only give them access to the media player and nothing else.

This could be accomplished by running the components on seperate systems altogether, however that is inefficient and will most likely be a waste of resources. Instead, we can run each component in a virtual environment, monitored by a hypervisor such as Xen or KVM. We will be working with Xen due to several security advantages.

Xen is a type 1 hypervisor meaning that it runs directly on the hardware and does not rely on any underlying operating system. Xen works by booting directly on the hardware and then creating multiple domains (called dom's), each of which contains its own operating system. The only required domain is known as Dom0 and is the domain that will hold your device drivers and manage the other guests. Any other domain is refered to as a DomU (Dom1, Dom2, etc.) Traditionally, Dom0 boots first and then it is what boots any DomU's. But this leads to the question, what if Dom0 is compromised? Thanks to some tricks with device trees, it is now possible to boot Xen as Dom0-less. This does not mean that we dont have Dom0 but it means that safety critical DomU's can be booted by Xen directly rather than by Dom0. In addition to better safety, this also improves the boot time of your DomU's.

But what if the safety critical system uses one of the device drivers that is in Dom0? Xen has a solution for this too! We can create a special type of domain called a "driver domain" which will be be isolated and only provide driver functionality to the other domains. Now, even if Dom0 goes down or is corrupted, our DomU's still have access to the drivers without having to interact with Dom0. While the main goal of this is to improve the isolation, it also adds the ability for a DomU to use a driver that is not supported by Dom0.

## 0.2 - Attack Surface

The attack surface of a system is the sum of the different points of access where an attacker can retrieve or insert data. When worying about safety, it is always important to try to eliminate or reduce the attack surface as much as possible. You can imagine this like a house with a security system and sensors on every door. A house with no door will be easy to secure with just no sensors but if you have 10 doors, you have a lot more entry points to worry about. What we want is to find a balance between these two where we still get all of the functionality we want but there are no unnecessary entrances. Maybe 1 or 2 doors.

So why is it that to run something such as a headless web server, we are installing so many unused libraries? To accomplish this, peple will typically download a Linux distro and then use that to boot somethinng like Node.js or NGINX. This means that even if we know our application is safe, we have to rely on the security of the operating system and every one of its dependencies. Instead of this, we can introduce the idea of a Unikernel.

A Unikernel is a "specialised, single-address-space machine image constructed by using library operating systems." This means that instead of having the seperation of kernel and user space, we can hand pick which libraries we want and build our application directly into the kernel.

## 0.3 - Application

![Xen Diagram](/Users/alexanderclarke/dev/Unikernel-Exploration/images/Xen Diagram.png)

In order to explore these concepts and technologies, I have been tasked with creating a demo on the Raspberry Pi 4b. The Raspberry Pi was chosen, not only for its capabilities, but also for its popularity. The Pi has quickly become one of the defacto boards for cheap home experimentation and I believe that it is the most accessible platform for most people to work with. When going into this project, I was mostly unfamiliar with Xen, Unikernels, Arm architecture, and Linux but I have tried to highlight all of the key components that are going into our final demo and show you how to build them for yourself.

