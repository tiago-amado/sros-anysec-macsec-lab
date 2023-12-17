
# CLAB SROS FP5 ANYSec and MACSec Demo

ANYSec is a Nokia technology that provides low-latency and line-rate native encryption for any transport (IP, MPLS, segment routing, Ethernet or VLAN), on any service, at any time and for any load conditions without impacting performance.

This lab is an ANYSec demo using Nokia SROS FP5 (https://www.nokia.com/networks/technologies/fp5/) vSIMs running at CLAB (https://containerlab.dev/).
It combines ANYSec with MACSec and ilustrates ANYSec slicing for distinct services with multi-instance SR-ISIS and FLEX-Algo. 
It also provides a visiluazitation dashboard using a Telemetry stack with gNMIc, Prometheus and Grafana.
The automation uses Flask with python and gNMIc to start/stop traffic, enable/disable links and enable/disable ANYSec.




## ANYSec Overview
ANYSec is a Nokia network encryption solution available with the new FP5 models in SROS 23.10R1. 
It is low-latency line-rate encryption, scalable, flexible and ensures a quantum-safe network encryption solution for the industry.
It is a simple concept, based on MACSec standards as the foundation and introduces the flexibility to offset the authentication and encription to allow L2, L2.5 and L3 encryption.



## Clone the git lab to your server

To deploy these labs, you must clone these labs to your server with "git clone".

```bash
# change to your working directory
cd /home/user/
# Clone the lab to your server
git clone https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec.git
```


## SROS Image and License file

### SROS Image

The SROS vSIMs image file used is 23.10R2, and is available under Nokia's internal registry. 
If you don't have access to it, then you must get the SROS image and manually import them to CLAB following the instructions here: https://containerlab.dev/manual/vrnetlab/#vrnetlab

The stepts are:
```bash
# Clone vrnetlab
git clone https://github.com/hellt/vrnetlab && cd vrnetlab

# Download qcow2 vSIM image from Nokia support portal (https://customer.nokia.com/support/s) or get one from your Nokia contact. 

# Change name to “sros-vm-<VERSION>.qcow2”   ### must start with "sros-vm-"

# Upload it to ‘vrnetlab/sros’ directory (e.g. /home/vrnetlab/sros)

# Run ‘make docker-image’ to start the build process

# Verify existing docker images

docker images | grep -E "srlinux|vr-sros"
```

Note: After import the image, edit the yml file with the correct location.
```bash
# replace this 
      image: registry.srlinux.dev/pub/vr-sros:23.10.R2
# with this:
      image: vrnetlab/vr-sros:23.10.R2
```

### License file

SROS vSIMs require a valid license. You need to get a valid license from Nokia and place it in the "/r23_license.key" file.
```bash
# Copy/paste the license to the "r23_license.key" file
cd SROS_CLAB_FP5_ANYSec/
vi r23_license.key
# press "i" key for insert mode => paste the license => ctl+x to save and exit 
```



## ANYSec setup


### Physical setup

The physical setup is ilustrated below:



<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/physical-setup.jpg?raw=true">
</p>



The setup contains six SROS FP5 routers with 23.10R2 and 2 linux. The network contains 2 P routers, 2 PEs running ANYSec and MACSec, 2 CEs with MACSec, and 2 Linux Clients with 3 interfaces for distinct services. 
howhever only two of them have ANYSec configured:
   - P Routers
		•	sr-2se (FP5 only)
		•	sr-7s (FP5 only)
   - PE Routers with ANYSec and MACSec
		•	sr-1x-92s 
		•	sr-1se
   - CE Routers with MACSec
		•	sr-1-24d
		•	sr-1-46s
   - Clients are Linux hosts





### Logical setup



The logical setup with the services is the following:



<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/vprn.jpg?raw=true">
</p>


The setup has:

•	ANYSec between PE1 and PE2

•	MACSec between PEs and CEs

•	ISIS instances 0, 1 and 2 with SR-ISIS and Flex-Algo

•	iBGP (P3 and P4 as RR)

•	Services: VLL 1001, VPLS 1002 and VPRN 1003




### Services and Slicing

ANYSec slicing is possible within 20.10R1 with 2 options:
•	Multi-Instance SR IGP instance
•	Flex-Algo


To demonstrate both options, 3 ISIS instance are configured:
•	ISIS 0 – Flex-Algo with TE-Metrics (other constraints are possible)
•	ISIS 1- IGP metrics to prefer TOP LINK
•	ISIS 2 – IGP metrics to prefer BOTTOM LINK


Regarding Services, also 3 are created:
•	VLL 1001 – ISIS 1 => TOP LINK
•	VPLS 1002 – ISIS 2 => BOTTOM LINK
•	VPRN 1003 – ISIS 0 => Flex-Algo 


Note: Each of the 3 client interfaces is mapped to a distinct service. Its possible to start iPerf or ICMP between on every interface to test the distinct topologies.


The 3 SR-ISIS topologies are illustrated bellow:



<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/vll.jpg?raw=true">
</p>







## Deploy the lab setup

Use the comand below to deploy the lab:

Note: If you imported the SROS image to docker then first edit the yml file with the correct image location as explained above.

```bash
# deploy a lab
cd sros-anysec-macsec-lab/
clab deploy --topo anysec-macsec.yml
```



## Accessing the network elements

Once the lab is deployed, the different SROS nodes can be accessed via SSH through their management IP address, given in the summary displayed after the execution of the deploy command. 
It is also possible to reach those nodes directly via their hostname, defined in the topology file. 

```bash
# List the containers
clab inspect -a
# reach a SROS node via SSH
ssh admin@pe1
# reach Linux clients via docker
docker exec -it client7 bash
```


## Wireshark

For details about Packet capture & Wireshark at containerlab refer to:
https://containerlab.dev/manual/wireshark/#capturing-with-tcpdumpwireshark


You may found a pcap file with ANYSec packets in the files above in this project. 
You may perform your own capture as explained below.


### Local capture 

Follows an example on how to list the interfaces (links) of a given container and perform a packet capture:
```bash
# list the containers running in the server
clab inspect -a 
# list the interfaces (links) of a given container
ip netns exec pe1 ip link
# Start a capture and display packets in the session
ip netns exec pe1 tcpdump -nni eth1
# Start a capture and store the packets in the file
ip netns exec pe1 tcpdump -nni eth1 -w capture_file.pcap
```


### Remote capture 

Besides displaying the packets to the session or store in a file, its possible to open them directly on Wireshark using a remote SSH connection.


Follows examples of the SSH comand from a Linux Shell or Windows Comand Prompt:
```bash
Syntax:
ssh $containerlab_host_address "ip netns exec $lab_node_name tcpdump -U -nni $if_name -w -" | $wireshark_path -k -i -

Linux example:
ssh root@10.82.182.179 "ip netns exec pe1 tcpdump -U -nni eth1 -w -" | /mnt/c/Program\ Files/Wireshark/wireshark.exe -k -i -
Windows example:
ssh root@10.82.182.179 "ip netns exec pe1 tcpdump -U -nni eth1 -w -" | "c:\Program Files\Wireshark\Wireshark.exe" -k -i -
```



## SROS Streaming Telemetry and Automation

This lab was enhanced with Streaming Telemetry by adding gNIMc, Prometheus and Grafana.

For details please refer to: 
https://github.com/srl-labs/srl-sros-telemetry-lab


It also includes Automation for the tests using gNMIC invoked through python from Flask webserver. There are 3 main set of tests:

1 - Start/Stop traffic for each service

2 - Disable/enable the top link (between PE1 and P3) to see ANYSec packets flowing through the bottom link.

3 - Disable/enable ANYSec to see packets being sent in clear or encrypted on demand

 


### Telemetry and automation stack

The following stack of software solutions has been chosen for this lab:

| Role                | Software                                            | Port               | Link                               | Credentials        |
| ------------------- | --------------------------------------------------- |------------------- | ---------------------------------- |------------------- |
| Telemetry collector | [gnmic](https://gnmic.openconfig.net)               | 57400              |                                    |                    |
| Time-Series DB      | [prometheus](https://prometheus.io)                 | 9090               | http://localhost:9090/graph        |                    |
| Visualization       | [grafana](https://grafana.com)                      | 3000               | http://localhost:3000              | admin/admin        |
| Flask Server/gnmic  | [flask](https://flask.palletsprojects.com/en/3.0.x/)| 9080               | http://localhost:9080/             |                    |



The following picture picture ilustrates the Telemetry and Automation stack:



<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/telemetry_automation.jpg?raw=true">
</p>




### Access details

If you are accessing from a remote host, then replace localhost by the CLAB Server IP address
* Grafana: <http://localhost:3000>. Built-in user credentials: `admin/admin`
* Prometheus: <http://localhost:9090/graph>
* Flask Demo Page: <http://localhost:9080/>   




## Verify the setup

Verify that you're able to access all nodes (Routers and clients) and the platforms (Grafana, Prometheus and Flask Demo Page).

Start a Tcpdump/wireshark capture as explained above and start traffic between Client7 and Client8 using Flask. 

You may shut the link between PE1 and P3 and see that ANYSec SR-ISIS traffic uses the bottom link.

You may also disable ANYSec to view packets in clear.



### Wireshark ANYSec Decoding


Wireshark does not have native support for decoding ANYSec MACSec (802.1AE) headers. You'll be able to see the MPLS and ANYSec labels, but the MACSec header will not be decoded. 

Nokia has an internal version with a protocol dissector for ANYSec MACSec / 802.1a headers.
This is the output comparison between the public wireshark and the Nokia's version:


![pic1](https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/ANYSec_Wireshark.jpg)


Note: With the public Wireshark, the ANYSec header not decoded but you still be able to validate ANYSec looking into the ANYSec label.




### ANYSec Stack


The ANYSec introduces the MACSec Header and the Encryption SID (ES) label between the SR-ISIS transport and VPRN service labels. The VPRN service label is encrypted.
The picture below provides an example of the ANYSec label stack between R1 and R2.


![pic1](https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/ANYSec_Stack.jpg?raw=true)




### Capture multiple interfaces 


TCPDUMP on a single interface shows label stack correctly (Ethernet+VLAN+MPLS+ANYSec)
TCPDUMP on a multiple interfaces shows a distinct stack: Linux cooked capture v2 + additional MPLS Label (instead of Ethernet + VLAN)


![pic1](https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/ANYSec_Tcpdump.jpg?raw=true)





## Outputs

Use the following commands under PE1 or PE2 to retrieve outputs from ANYSec operation:


```bash
show macsec connectivity-association "CA_Test_MACSec" detail 
show anysec tunnel-encryption detail 
show router 1003 route-table 10.0.0.2/32 extensive 
show router tunnel-table detail 
show router mpls-labels summary 
show router "1003" route-table 
show router bgp routes 10.0.0.2/32 vpn-ipv4 hunt   
```



## Tests

The tests bellow can be executed in multiple ways: flask demo page, gnmic scripts or node CLI.


### Test 1 - Shut/No shut the link between R1 and R2 

Upon shut/no shut verify ANYSec is still working but using a new SR-ISIS tunnel
```bash
show router "1003" route-table
show router 1003 route-table 10.0.0.2/32 extensive
show router 1003 route-table 10.0.0.2/32 extensive
show router bgp routes 10.0.0.2/32 vpn-ipv4 hunt   
```




<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/LINK-DOWN.jpg?raw=true">
</p>

### Test 2 - Disable ANYSec at R1 and R2 

Upon Disable ANYSec verify ping is still working but unecripted
Re-enable ANYSec and verify traffic is encrypted again




<p align="center">
  <img width="900" height="500" src="https://github.com/tiago-amado/SROS_CLAB_FP5_ANYSec/blob/main/pics/ANYSec-DISABLE.jpg?raw=true">
</p>





## ANYSec Demo Video

The Demo Video shows the Grafana Dashboard, the wireshark and the CLI with ICMP. Two tests are performed: disable/enable the top link and disable/enable ANYSec.


[![Watch the video](http://img.youtube.com/vi/Ka6-zXaPYGI/maxresdefault.jpg)](https://youtu.be/Ka6-zXaPYGI)






## Conclusion

ANYSec is an amazing technology, flexible and scalable, capable of E2E low-latency and line-rate transport encryption.
ANYSec can be combined with other technologies such as MACSec or IPSec. It allows slicing and per service encryption.


Does ANYSec work with CLAB vSIMs?
Yes for functional tests, but obviously not for performance/latency.
CLAB and vSIMs can be used to test and validate the configurations. 
Setup is fully functional with ANYSec stats increase and packets are encrypted as seen in the TCPDUMP capture.

More to come in the upcoming releases!

