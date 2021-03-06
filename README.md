This builds Windows 2012R2/10/2016/2019 base Vagrant boxes using [Packer](https://www.packer.io/).


# Usage

Install [VirtualBox](https://www.virtualbox.org/) (or [libvirt](https://libvirt.org/) on Linux based systems), [packer](https://www.packer.io/), [packer-provisioner-windows-update plugin](https://github.com/rgl/packer-provisioner-windows-update) and [vagrant](https://www.vagrantup.com/).
If you are using Windows and [Chocolatey](https://chocolatey.org/), you can install everything with:

```batch
choco install -y virtualbox packer packer-provisioner-windows-update vagrant
```

To build the base box based on the [Windows Server 2016 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2016) ISO run:

```bash
make build-windows-2016-libvirt # or make build-windows-2016-virtualbox
```

If you want to use your own ISO, you need to manually run the `packer` command, e.g.:

```bash
packer build -var iso_url=<ISO_URL> -var iso_checksum=<ISO_SHA256_CHECKSUM> -only=windows-2016-amd64-virtualbox windows-2016.json
```

**NB** if the build fails with something like `Post-processor failed: write /tmp/packer073329394/packer-windows-2016-amd64-virtualbox-1505050546-disk001.vmdk: no space left on device` you need to increase your temporary partition size or change its location [as described in the packer TMPDIR/TMP environment variable documentation](https://www.packer.io/docs/other/environment-variables.html#tmpdir).

**NB** if you are having trouble building the base box due to floppy drive removal errors try adding, as a
workaround, `"post_shutdown_delay": "30s",` to the `windows-2016.json` file.

**NB** the packer logs are saved inside a `*-packer.log` file (e.g. `windows-2016-amd64-libvirt-packer.log`).

You can then add the base box to your local vagrant installation with:

```bash
vagrant box add -f windows-2016-amd64 windows-2016-amd64-virtualbox.box
```

And test this base box by launching an example Vagrant environment:

```bash
cd example
vagrant plugin install vagrant-windows-sysprep
vagrant up --provider=virtualbox # or --provider=libvirt
```

**NB** if you are having trouble running the example with the vagrant libvirt provider check the libvirt logs in the host (e.g. `sudo tail -f /var/log/libvirt/qemu/example_default.log`) and in the guest (inside `C:\Windows\Temp`).

Then test with a more complete example:

```bash
git clone https://github.com/rgl/customize-windows-vagrant
cd customize-windows-vagrant
vagrant up --provider=virtualbox # or --provider=libvirt
```


## libvirt

Build the base box for the [vagrant-libvirt provider](https://github.com/vagrant-libvirt/vagrant-libvirt) with:

```bash
make build-windows-2016-libvirt
```

If you want to access the UI run:

```bash
spicy --uri 'spice+unix:///tmp/packer-windows-2016-amd64-libvirt-spice.socket'
```

**NB** the packer template file defines `qemuargs` (which overrides the default packer qemu arguments), if you modify it, verify if you also need include the default packer qemu arguments (see [builder/qemu/step_run.go](https://github.com/hashicorp/packer/blob/master/builder/qemu/step_run.go) or start packer without `qemuargs` defined to see how it starts qemu).


## VMware vSphere

Download `packer-builder-vsphere-iso` v2.3 from the [jetbrains-infra/packer-builder-vsphere releases page](https://github.com/jetbrains-infra/packer-builder-vsphere/releases) and place it inside your `~/.packer.d/plugins` directory (on Windows its at `%USERPROFILE%\packer.d\plugins` or `%APPDATA%\packer.d\plugins`).

Download the Windows Evaluation ISO (you can find the full iso URL in the [windows-2016-vsphere.json](windows-2016-vsphere.json) file) and place it inside the datastore as defined by the `vsphere_iso_url` user variable that is inside the [packer template](windows-2016-vsphere.json).

Download the [VMware Tools VMware-tools-windows-&lt;SAME_VERSION_AS_IN_PACKER_TEMPLATE&gt;.iso](https://packages.vmware.com/tools/releases/index.html) file into the datastore defined by the `vsphere_tools_iso_url` user variable that is inside the [packer template](windows-2016-vsphere.json).

Download [govc](https://github.com/vmware/govmomi/releases/latest) and place it inside your `/usr/local/bin` directory.

Install the vsphere vagrant plugin, set your vSphere details, and test the connection to vSphere:

```bash
sudo apt-get install build-essential patch ruby-dev zlib1g-dev liblzma-dev
vagrant plugin install vagrant-vsphere
vagrant plugin install vagrant-windows-sysprep
cd example
cat >secrets.sh <<'EOF'
export GOVC_INSECURE='1'
export GOVC_HOST='vsphere.local'
export GOVC_URL="https://$GOVC_HOST/sdk"
export GOVC_USERNAME='administrator@vsphere.local'
export GOVC_PASSWORD='password'
export GOVC_DATACENTER='Datacenter'
export GOVC_CLUSTER='Cluster'
export GOVC_DATASTORE='Datastore'
export VSPHERE_ESXI_HOST='esxi.local'
export VSPHERE_TEMPLATE_FOLDER='test/templates'
# NB the VSPHERE_TEMPLATE_NAME last segment MUST match the
#    builders.vm_name property inside the packer tamplate.
export VSPHERE_TEMPLATE_NAME="$VSPHERE_TEMPLATE_FOLDER/windows-2019-amd64-vsphere"
export VSPHERE_TEMPLATE_IPATH="//$GOVC_DATACENTER/vm/$VSPHERE_TEMPLATE_NAME"
export VSPHERE_VM_FOLDER='test'
export VSPHERE_VM_NAME='windows-2019-vagrant-example'
export VSPHERE_VLAN='packer'
EOF
source secrets.sh
# see https://github.com/vmware/govmomi/blob/master/govc/USAGE.md
govc version
govc about
govc datacenter.info # list datacenters
govc find # find all managed objects
```

Build the base box with:

```bash
make build-windows-2019-vsphere
```

Try the example guest:

```bash
source secrets.sh
echo $VSPHERE_TEMPLATE_NAME # check if you are using the expected template.
vagrant up --provider=vsphere
vagrant ssh
exit
vagrant destroy -f
```


## WinRM access

You can connect to this machine through WinRM to run a remote command, e.g.:

```batch
winrs -r:localhost:55985 -u:vagrant -p:vagrant "whoami /all"
```

**NB** the exact local WinRM port should be displayed by vagrant, in this case:

```plain
==> default: Forwarding ports...
    default: 5985 (guest) => 55985 (host) (adapter 1)
```


# WinRM and UAC (aka LUA)

This base image uses WinRM. WinRM [poses several limitations on remote administration](http://www.hurryupandwait.io/blog/safely-running-windows-automation-operations-that-typically-fail-over-winrm-or-powershell-remoting),
those were worked around by disabling User Account Control (UAC) (aka [Limited User Account (LUA)](https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/microsoft-windows-lua-settings-enablelua)) in `autounattend.xml`
and [UAC remote restrictions](https://support.microsoft.com/en-us/help/951016/description-of-user-account-control-and-remote-restrictions-in-windows)
 in `winrm.ps1`.

If needed, you can later enable them with:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name EnableLUA -Value 1
Set-ItemProperty -Path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Policies\System' -Name EnableLUA -Value 1
Remove-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy
Restart-Computer
```

Or disable them with:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name EnableLUA -Value 0
Set-ItemProperty -Path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Policies\System' -Name EnableLUA -Value 0
New-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Force
Restart-Computer
```


# Windows Unattended Installation

When Windows boots from the installation media its Setup application loads the `a:\autounattend.xml` file.
It contains all the answers needed to automatically install Windows without any human intervention. For
more information on how this works see [OEM Windows Deployment and Imaging Walkthrough](https://technet.microsoft.com/en-us/library/dn621895.aspx).

`autounattend.xml` was generated with the Windows System Image Manager (WSIM) application that is
included in the Windows Assessment and Deployment Kit (ADK).

## Windows ADK

To create, edit and validate the `a:\autounattend.xml` file you need to install the Deployment Tools that
are included in the [Windows ADK](https://developer.microsoft.com/en-us/windows/hardware/windows-assessment-deployment-kit).

If you are having trouble installing the ADK (`adksetup`) or running WSIM (`imgmgr`) when your
machine is on a Windows Domain and the log has:

```plain
Image path is [\??\C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\DISM\wimmount.sys]
Could not acquire privileges; GLE=0x514
Returning status 0x514
```

It means there's a group policy that is restricting your effective permissions, for an workaround,
run `adksetup` and `imgmgr` from a `SYSTEM` shell, something like:

```batch
psexec -s -d -i cmd
adksetup
cd "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\WSIM"
imgmgr
```

For more information see [Error installing Windows ADK](http://blogs.catapultsystems.com/chsimmons/archive/2015/08/17/error-installing-windows-adk/).
