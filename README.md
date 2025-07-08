# Azure Windows VMs Active Directory Lab

This guide walks through installing Active Directory on a Domain Controller VM (DC-1), creating users and OUs, joining a Client VM (Client-1) to the domain, and configuring Remote Desktop for non-admin users.  

---

## Prerequisites

- Two Azure VMs already created:
  - **DC-1** (Windows Server, intended as Domain Controller)
  - **Client-1** (Windows 10/11, intended as domain-joined client)
- Both VMs in the same Virtual Network/subnet.
- `labuser` account credentials on both VMs.
- Azure Portal access to start/stop VMs and configure networking.

---

## Part 1 – Install AD DS & Create Domain Accounts

### 1. Start VMs
1. In the Azure Portal, navigate to each VM (DC-1 and Client-1).
2. Click **Start** if either VM is stopped.

### 2. Install Active Directory Domain Services on DC-1
1. **RDP** into **DC-1** as `.\labuser`.
2. Open **Server Manager** → **Add roles and features**.
3. Select **Active Directory Domain Services** and install.

https://github.com/user-attachments/assets/eb9078c7-bcf1-4e78-8c6c-b284b3762ba5
   
4. After installation, click the **flag** icon in Server Manager and choose **Promote this server to a domain controller**.
5. Choose **Add a new forest**, set **Root domain name** to `mydomain.com` (or your preferred name).
6. Complete the wizard and allow the server to **restart**.


https://github.com/user-attachments/assets/ad78bfdf-86c6-423d-9f59-d9b56b2703fb


### 3. Create Domain Admin User
1. After reboot, log in as `mydomain.com\labuser` (password: `Cyberlab123!`).
2. Open **Active Directory Users and Computers (ADUC)**.
3. Right-click your domain → **New → Organizational Unit**, name it `_EMPLOYEES`.
4. Right-click your domain → **New → Organizational Unit**, name it `_ADMINS`.
5. Under the `_ADMINS` OU, **New → User**:
   - **First name**: Jane  
   - **Last name**: Doe  
   - **User logon name**: `jane_admin`  
   - **Password**: `Cyberlab123!`  
6. Finish the wizard, then add `jane_admin` to the **Domain Admins** group:
   - Double-click `jane_admin` → **Member Of** tab → **Add…** → enter `Domain Admins`.
7. Sign out, then **RDP** back into **DC-1** as `mydomain.com\jane_admin`.


https://github.com/user-attachments/assets/660ef55f-5f0b-4abe-9761-2f6fec993660


### 4. Join Client-1 to the Domain
1. Ensure **Client-1** DNS is pointed at **DC-1**’s private IP (already done).
2. In the Azure Portal, restart **Client-1**.
3. RDP into **Client-1** as the local admin (`.\labuser`).
4. Open **System Properties** → **Computer Name** → **Change…**.
5. Select **Domain**, enter `mydomain.com`, click **OK**.
6. Provide `mydomain.com\jane_admin` credentials when prompted.
7. Allow **Client-1** to restart to complete the join.
8. On **DC-1**, open **ADUC**, verify **Client-1** appears in **Computers**.
9. In ADUC, create a new OU named `_CLIENTS`.
10. Drag **Client-1** object into the `_CLIENTS` OU.

> **Tip:** When you’re done for now, you can stop both VMs in the Azure Portal to save costs.

---

## Part 2 – Remote Desktop & Bulk User Creation

### 1. Enable RDP for Domain Users on Client-1
1. In the Azure Portal, **start** DC-1 and Client-1 if they’re off.
2. RDP into **Client-1** as `mydomain.com\jane_admin`.
3. Open **System Properties** → **Remote** tab → **Select** “Allow remote connections to this computer”.
4. Click **Select Users…** → **Add…** → type `Domain Users` → **OK**.
5. All domain users can now RDP into Client-1 (no admin rights needed).

### 2. Bulk-Create Users via PowerShell Script
1. RDP into **DC-1** as `mydomain.com\jane_admin`.
2. Open **Windows PowerShell ISE** **as Administrator**.
3. **File → New**, paste your user-creation script, for example:
   ```powershell
   # Example Bulk User Creation Script
   Import-Module ActiveDirectory

   $users = @(
     @{Name='Alice Smith'; SamAccountName='alice.smith'; OU='_EMPLOYEES'; Password='P@ssw0rd1'},
     @{Name='Bob Jones';   SamAccountName='bob.jones';   OU='_EMPLOYEES'; Password='P@ssw0rd1'},
     # … add more entries here …
   )

   foreach ($u in $users) {
     New-ADUser `
       -Name         $u.Name `
       -SamAccountName $u.SamAccountName `
       -AccountPassword (ConvertTo-SecureString $u.Password -AsPlainText -Force) `
       -Enabled      $true `
       -Path         "OU=$($u.OU),DC=mydomain,DC=com" `
       -PasswordNeverExpires $false
   }
