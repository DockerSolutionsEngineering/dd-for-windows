<!-- Output copied to clipboard! -->

<!-----

Yay, no errors, warnings, or alerts!

Conversion time: 0.327 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β34
* Thu Aug 03 2023 06:07:25 GMT-0700 (PDT)
* Source doc: Docker Desktop Hyper-V Security
----->



# Docker Desktop Hyper-V Security


# Purpose of this document

This document describes the security-related aspects of the Hyper-V implementation in Docker Desktop.


### Windows Hyper-V implementation details

On Windows the Docker Desktop Service listens on a named pipe `//./pipe/dockerBackendV2`. The developer runs the Docker Desktop application, which connects to the named pipe and sends commands to the service.

This named pipe is protected, and only users that are part of the docker-users group can have access to it. It provides a very minimal subset of Hyper-V commands like create/start/stop/destroy DockerDesktopVM. The VM name is hard coded in the service code, so the user cannot create or manipulate any other VM.

The only parameters that can be configured through those APIs are number of CPUs, allocated memory size, and the disk location (disallowing non-sensible system locations).

The Docker Desktop VM is a minimal, special-purpose VM. It is immutable, meaning that users can’t extend it or change the installed software. Even if a user runs a privileged container, they will not be able to alter the VM, let alone escape from the VM to change anything on the host.

For network connectivity, Docker Desktop uses a user-space process (vpnkit), which inherits constraints like firewall rules, VPN, http proxy properties etc. from the user that launched it.

Similarly, file sharing (bind mount from the host filesystem) uses a user-space crafted file server (running in com.docker.backend), so the VM can’t access any files other than those the user already has access to.
