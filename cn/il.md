# Overview
## 1.1  Introduction
This document details the Application Programming Interface (API) for the OpenMAX
Integration Layer (IL). Developed as an open standard by The Khronos Group, the IL
serves as a low-level interface for audio, video, and imaging codecs used in embedded
and/or mobile devices. The principal goal of the IL is to give codecs a degree of system
abstraction for the purpose of portability across operating systems and software stacks.
## 1.1.1 About the Khronos Group
The Khronos Group is a member-funded industry consortium focused on the creation of
open standard APIs to enable the authoring and playback of dynamic media on a wide
variety of platforms and devices. All Khronos members may contribute to the
development of Khronos API specifications, may vote at various stages before public
deployment, and may accelerate the delivery of their multimedia platforms and
applications through early access to specification drafts and conformance tests. The
Khronos Group is responsible for open APIs such as OpenGL ES, OpenML, and
OpenVG.
## 1.1.2 A Brief History of OpenMAX
The OpenMAX set of APIs was originally conceived as a method of enabling portability
of codecs and media applications throughout the mobile device landscape. Brought into
the Khronos Group in mid-2004 by a handful of key mobile hardware companies,
OpenMAX has gained the contributions of companies and institutions stretching the
breadth of the multimedia field. As such, OpenMAX stands to unify the industry in
taking steps toward media codec portability. Stepping beyond mobile platforms, the
general nature of the OpenMAX IL API makes it applicable to all media platforms.
1.2  The OpenMAX Integration Layer
The OpenMAX IL API strives to give media codecs portability across an array of
platforms. The interface abstracts the hardware and software architecture in the system.
Each codec and relevant transform is encapsulated in a component interface. The
OpenMAX IL API allows the user to load, control, connect, and unload the individual
components. This flexible core architecture allows the Integration Layer to easily
implement almost any media use case and mesh with existing graph-based media
frameworks.
1.2.1 Key Features and Benefits
The OpenMAX IL API gives applications and media frameworks the ability to interface
with multimedia codecs and supporting components (i.e., sources and sinks) in a unified
manner. The codecs themselves may be any combination of hardware or software and
are completely transparent to the user. Without a standardized interface of this nature,
codec vendors must write to proprietary or closed interfaces to integrate into mobile