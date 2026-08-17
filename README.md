# BUIKE-HELPDESK (osticket.local)

<p align="center">
  <img src="images/osticket-logo-banner.png" alt="osTicket logo" width="100%" />
</p>

A self-hosted support ticketing system built on Azure. Provisioning, web server configuration, PHP and MySQL setup, and application deployment, all done by hand and documented like a production rollout.

<p align="center">
  <img src="images/03-vm-networking.png" alt="Azure VM provisioning" width="58%" />
  <img src="images/25-congratulations.png" alt="osTicket install complete" width="33%" />
</p>

<p align="center"><em>Left: creating the osticket-vm virtual machine, setting the virtual network, subnet, and public IP on the Networking tab, inside the OS-Ticktet-RG resource group. Right: the osTicket Installer's Congratulations screen, confirming osTicket v1.15.8 installed successfully, with links to the live helpdesk and the staff control panel.</em></p>

---

## Why This Project Exists

I wanted hands-on practice standing up a real application on real infrastructure instead of clicking through a cloud trial. This build takes a Windows Server VM in Azure from a blank resource group all the way to a working help desk: IIS installed and configured by hand, PHP wired in through PHP Manager, MySQL set up as the backend, and osTicket deployed and installed through its own setup wizard.

It also follows real operational habits. Every resource is named and tracked, every dependency is documented, and the one real problem I hit along the way, an HTTP 500 error from a permissions issue, is written up with what caused it and how it got fixed.

---

## Environment

| Asset ID | Resource | Role | Details |
|---|---|---|---|
| RG-OST01 | OS-Ticktet-RG | Resource group | East US 2 |
| VM-OST01 | osticket-vm | Application host | Windows Server, running IIS, PHP, and MySQL |
| NET-OST01 | vnet-eastus2-1 | Virtual network | 172.16.0.0/24 |
| IP-OST01 | osticket-vm-ip | Public IP | Attached to VM-OST01 |
| NSG-OST01 | osticket-vm-nsg | Network security group | Basic rule set |

Everything lives in a single resource group so the whole build can be torn down or redeployed cleanly. Remote Desktop is the only way in, there is no separate jump box or bastion for a build this size.

---

## Software Stack

| Component | Version | Purpose |
|---|---|---|
| IIS | 10 | Web server |
| PHP | 7.3.8 | Application runtime |
| PHP Manager for IIS | 1.5.0 | Registers and manages PHP inside IIS |
| IIS URL Rewrite Module | 2 | Clean URL routing, required by osTicket |
| VC++ Redistributable | 2015-2022 (x86) | Runtime dependency for PHP and MySQL |
| MySQL Server | 5.5 | Database backend, Standard Configuration |
| HeidiSQL | 12.3 | Database management client |
| osTicket | v1.15.8 | Help desk application |

---

## Installation Walkthrough

### 1. Provision the virtual machine

Everything starts in the Azure Portal. I created a dedicated resource group, OS-Ticktet-RG, so every piece of this build, the VM, the network, the public IP, would live in one place and be easy to clean up later if needed.

<p align="center"><img src="images/01-resource-group.png" width="80%"></p>
<p align="center"><img src="images/02-resource-group-review.png" width="80%"></p>

Once the resource group was set up, I created the actual virtual machine. On the networking tab I let Azure generate a new virtual network and subnet, added a public IP so I could reach the box from outside, and left the network security group on the basic setting for now.

<p align="center"><img src="images/03-vm-networking.png" width="80%"></p>
<p align="center"><img src="images/04-vm-deployment.png" width="80%"></p>

### 2. Grab the installation files

With the VM deployed, I remoted into it and downloaded a zip bundle containing everything needed: osTicket itself, PHP, PHP Manager for IIS, the URL Rewrite Module, the Visual C++ Redistributable, HeidiSQL, and MySQL. Extracting the zip pulls all of that out into one folder so Windows can run the installers inside it.

<p align="center"><img src="images/05-extract-files.png" width="80%"></p>
<p align="center"><img src="images/06-installation-files.png" width="80%"></p>

### 3. Turn on IIS and the CGI feature

This part lives in Control Panel. Open it from the Windows search bar, go to Programs, then Turn Windows features on or off. This is where Internet Information Services gets enabled. Underneath World Wide Web Services there is a subfolder called Application Development Features, and inside that is CGI, which needs to be checked. This is what lets IIS hand PHP requests off to the PHP engine.

<p align="center"><img src="images/07-enable-iis-cgi.png" width="80%"></p>

After enabling it, I confirmed IIS was actually running by browsing to the server's own address and seeing the default Windows welcome page.

<p align="center"><img src="images/08-iis-default-page.png" width="80%"></p>

### 4. Install PHP Manager and the URL Rewrite Module

PHP Manager for IIS is what lets IIS register and manage the PHP runtime, so that went in first.

<p align="center"><img src="images/09-php-manager-installed.png" width="80%"></p>

Right after that came the IIS URL Rewrite Module 2, which osTicket and most PHP applications need for clean, working URLs.

<p align="center"><img src="images/10-url-rewrite-setup.png" width="80%"></p>

### 5. Set up a home for PHP

Before installing PHP itself, I checked how much space was left on the C: drive, then created a plain folder at the root of C: called PHP to keep the runtime separate from the website files.

<p align="center"><img src="images/11-check-disk-space.png" width="80%"></p>
<p align="center"><img src="images/12-create-php-folder.png" width="80%"></p>

### 6. Install the Visual C++ Redistributable and MySQL

PHP and MySQL 5.5 both depend on the Microsoft Visual C++ 2015-2022 Redistributable, so that got installed next.

<p align="center"><img src="images/13-vcredist-install.png" width="80%"></p>

Then came MySQL Server 5.5 itself. I ran through the setup wizard and chose Standard Configuration, since this is a single machine that did not already have MySQL on it.

<p align="center"><img src="images/14-mysql-setup-wizard.png" width="80%"></p>
<p align="center"><img src="images/15-mysql-standard-config.png" width="80%"></p>

### 7. Drop the osTicket files into wwwroot

With the web stack in place, the osTicket application files went into IIS's web root at `C:\inetpub\wwwroot\osticket`, sitting right next to the PHP folder created earlier.

<p align="center"><img src="images/16-wwwroot-listing.png" width="80%"></p>
<p align="center"><img src="images/17-php-folder-verify.png" width="80%"></p>

### 8. Hit an HTTP 500 error on the first try

The first attempt to load the site did not go smoothly. Browsing to `localhost/osticket` threw an HTTP 500.0 Internal Server Error, with the FastCGI module reporting error code `0x80070003` while trying to run `index.php`. This is a classic sign of an NTFS permissions problem, meaning the IIS worker process did not have the access it needed to the application folder yet.

<p align="center"><img src="images/18-http-500-error.png" width="80%"></p>

### 9. Fix the folder permissions

To fix it, I opened the Advanced Security Settings on `ost-config.php`, the file inside `osticket\include` that stores the database connection details. At this point IIS_IUSRS only had Read and execute access, which was not enough.

<p align="center"><img src="images/19-permissions-advanced-security.png" width="80%"></p>

I added a new permission entry for the Everyone group and gave it Full control, so the installer would have the write access it needed to save the configuration during setup.

<p align="center"><img src="images/20-permission-entry-everyone.png" width="80%"></p>
<p align="center"><img src="images/21-select-user-or-group.png" width="80%"></p>

### 10. Run the osTicket installer

With permissions sorted, the browser based osTicket Installer finally loaded the way it was supposed to. The prerequisite check confirmed PHP 7.3.8 and the MySQLi extension were good to go, with only a handful of optional extensions like IMAP and Intl missing, which are not required for a basic setup.

<p align="center"><img src="images/22-installer-prereq-check.png" width="80%"></p>

From there I filled out the Basic Installation form: the helpdesk name, the admin account, and the database connection details pointing at the local MySQL instance and the osticket database.

<p align="center"><img src="images/23-basic-installation-form.png" width="80%"></p>

Hitting Install Now kicked off the actual installation.

<p align="center"><img src="images/24-installing-progress.png" width="80%"></p>

### 11. Success

And that was it. The installer finished with a clean Congratulations screen, confirming osTicket was installed and reminding me to lock down write access on `ost-config.php` once everything was configured. From here the helpdesk was reachable at `localhost/osticket`, with a staff control panel at `localhost/osticket/scp`.

<p align="center"><img src="images/25-congratulations.png" width="80%"></p>

---

## Things That Broke

- **HTTP 500 on the first load.** Browsing to `localhost/osticket` threw an Internal Server Error, with the FastCGI module reporting error code `0x80070003` while trying to run `index.php`. Root cause was NTFS permissions: `ost-config.php` was not writable by the IIS worker process. Fixed by opening Advanced Security Settings on the file and granting the Everyone group Full control, which let the installer write the config during setup.

---

## Build Phases

| Phase | Scope | Status |
|---|---|---|
| 1. Provisioning | Resource group, VM, and networking created in Azure | Complete |
| 2. Web Server | IIS installed, CGI feature enabled | Complete |
| 3. Runtime & Database | PHP, PHP Manager, URL Rewrite Module, VC++ Redistributable, MySQL | Complete |
| 4. Application Deploy | osTicket files copied into wwwroot | Complete |
| 5. Troubleshooting | HTTP 500 error resolved via NTFS permissions fix | Complete |
| 6. Installer | osTicket setup wizard completed end to end | Complete |
| 7. Hardening | Remove write access on ost-config.php, review permissions | Planned |

---

## Repo Structure

```
.
├── README.md
├── images/          Screenshots from every phase of the build
└── docs/            Phase writeups (planned)
```

---

*Built and documented by Chibuike "BK" Okerulu.*
