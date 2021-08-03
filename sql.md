

## Connecting to a remote SQL Server (on-prem)

This process uses Polybase and not linked servers, which means it should scale better.  

>The one thing I can't comment on is how to ensure connectivity between the AKS/BDC vnet and on-prem.  With ExpressRoute this should be a seamless experience with the Azure vnet appearing as just another subnet.  

For this documentation I set up a P2V VPN ([instructions](https://github.com/davew-msft/azure-P2S-vpn-automation)) to my local machine where I ran SQL Server in a container.  

Using wsl:

```bash
# start SQL Server container

docker pull mcr.microsoft.com/mssql/server:2019-CTP3.0-ubuntu

docker run -e 'ACCEPT_EULA=Y' \
   -e 'SA_PASSWORD=Password01!!' \
   -p 1433:1433 \
   -h DockSQL \
   --name DockSQL \
   -d \
   mcr.microsoft.com/mssql/server:2019-CTP3.0-ubuntu 


```

* Now, restore a database or create a database that we can connect to from BDC.
* use the ADS data virtualization wizard to connect to the sql server by IP address.  

I found that I needed to do basic network connectivity checks and open firewall ports.  

  * Since my docker container runs on 1433 I needed to ensure that my laptop firewall port was open.  
  * Probably the easiest way to ensure I could communicate from the vnet over the VPN was to spin up a small ubuntu vm where I could run `telnet local_machine_ip 1433` and then determine what additional firewall settings I needed to modify on the azure vnet.  
    * the alternative is to connect directly to the bash shell on one of the AKS nodes and run the commands there.  

