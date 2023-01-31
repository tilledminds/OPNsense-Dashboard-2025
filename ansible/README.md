run this using:
ansible-playbook -i inventory.yml -u root -k playbook.yml

It will prompt you for the SSH password to your OPNSense device.  This playbook assumes you've completed the first part of configuring OPNSense. 

Change the IP Address to whatever IP address you use for OPNSense
