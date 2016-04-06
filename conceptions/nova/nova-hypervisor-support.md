# Nova Hypervisor Support Matrix

以下說明目前 Nova 所支援的 Hypervisor 列表：

| Feature                            	| Status    	| Hyper-V 	| Ironic 	| Libvirt KVM (x86) 	| Libvirt LXC 	| Libvirt QEMU (x86) 	| Libvirt Xen 	| VMware vCenter 	| XenServer 	|
|------------------------------------	|-----------	|:-------:	|:------:	|:-----------------:	|:-----------:	|:------------------:	|:-----------:	|:--------------:	|:---------:	|
| Attach block volume to instance    	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Detach block volume from instance  	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Set the host in a maintenance mode 	| optional  	|    ✖    	|    ✖   	|         ✖         	|      ✖      	|          ✖         	|      ✖      	|        ✖       	|     ✔     	|
| Evacuate instances from a host     	| optional  	|    ?    	|    ?   	|         ✔         	|      ?      	|          ?         	|      ?      	|        ?       	|     ?     	|
| Guest instance status              	| mandatory 	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Guest host status                  	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Live migrate instance across hosts 	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✖       	|     ✔     	|
| Launch instance                    	| mandatory 	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Stop instance CPUs (pause)         	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✖       	|     ✔     	|
| Reboot instance                    	| optional  	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Rescue instance                    	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Resize instance                    	| optional  	|    ✔    	|    ✔   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Restore instance                   	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Service control                    	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✔       	|     ✔     	|
| Set instance admin password        	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✔     	|
| Save snapshot of instance disk     	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Suspend instance                   	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Swap block volumes                 	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Shutdown instance                  	| mandatory 	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Trigger crash dump                 	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✖     	|
| Resume instance CPUs (unpause)     	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✖       	|     ✔     	|
| Auto configure disk                	| optional  	|    ✔    	|    ✖   	|         ✖         	|      ✖      	|          ✖         	|      ✖      	|        ✖       	|     ✔     	|
| Instance disk I/O limits           	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✖     	|
| Config drive support               	| choice    	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Inject files into disk image       	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✔     	|
| Inject guest networking config     	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✔       	|     ✔     	|
| Remote desktop over RDP            	| choice    	|    ✔    	|    ✖   	|         ✖         	|      ✖      	|          ✖         	|      ✖      	|        ✖       	|     ✖     	|
| View serial console logs           	| choice    	|    ✔    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Remote interactive serial console  	| choice    	|    ✖    	|    ✖   	|         ✔         	|      ?      	|          ?         	|      ?      	|        ✖       	|     ✖     	|
| Remote desktop over SPICE          	| choice    	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✖     	|
| Remote desktop over VNC            	| choice    	|    ✖    	|    ✖   	|         ✔         	|      ✖      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Block storage support              	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Block storage over fibre channel   	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✖       	|     ✖     	|
| Block storage over iSCSI           	| condition 	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| CHAP authentication for iSCSI      	| optional  	|    ✔    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✖       	|     ✔     	|
| Image storage support              	| mandatory 	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Network firewall rules             	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✖       	|     ✔     	|
| Network routing                    	| optional  	|    ✖    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Network security groups            	| optional  	|    ✖    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| Flat networking                    	| choice    	|    ✔    	|    ✔   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| VLAN networking                    	| choice    	|    ✖    	|    ✖   	|         ✔         	|      ✔      	|          ✔         	|      ✔      	|        ✔       	|     ✔     	|
| uefi boot                          	| optional  	|    ✖    	|    ✔   	|         ✔         	|      ✖      	|          ✔         	|      ✖      	|        ✖       	|     ✖     	|
