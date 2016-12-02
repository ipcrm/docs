## Install the RUG CLI on Linux distributions

We support installing via packages on Debian- and RPM-based GNU/Linux
distributions.

### Debian/Ubuntu

To install on a Debian-based distributions, follow the next instructions:

1.  Grab the public GPG key for the repository:

        $ wget -qO - 'https://atomist.jfrog.io/atomist/api/gpg/key/public' | apt-key add

2.  Add a new apt source entry:

        $ echo 'deb https://atomist.jfrog.io/atomist/debian $(lsb_release -c -s) main' | sudo tee /etc/apt/sources.list.d/atomist.list"

3.  Update the metadata:

        $ sudo apt-get update

4.  Install the CLI:

        $ sudo apt-get install rug-cli

The only required dependency is the OpenJDK version 8 or later.

### RedHat/CentOS

To install on a RedHat-based distributions, follow the next instructions:

1.  Add a new yum repository:

        $ cat <<EOF | sudo tee /etc/yum.repos.d/atomist.repo
        [Atomist]
        name=Atomist
        baseurl=https://atomist.jfrog.io/atomist/yum/
        enabled=1
        gpgcheck=0
        EOF

2.  Install the CLI:

        $ sudo yum install rug-cli

The only required dependency is the JDK version 8 or later.