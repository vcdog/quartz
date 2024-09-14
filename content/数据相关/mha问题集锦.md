
# 问题一：

>**mha架构在主master出现问题，vip如果跨网段，但又归属于同一大vlan时，即使完成切换，vip访问不可达也会持续30-60s。在这个情况下，需要强制发起arping广播。**


# 解决办法：

在故障切换脚本中，定义变量 `$ssh_Bcast_arp` 引入ARP广播机制，在进行vip切换时，强制进行ARP广播。

```bash

# vim master_ip_online_change.3306

# 添加或修改如下这些数据行：

###########################################################################  
my $vip = '10.3.9.26/21';  
my $key = "1";
my $ssh_start_vip = "/sbin/ifconfig ens192:$key $vip";  
my $ssh_stop_vip = "/sbin/ifconfig ens192:$key $vip down";  
my $ssh_bcast_arp= "/sbin/arping -I ens192 -c 3 -A 10.3.9.26";
###########################################################################  


###############################################################################  
      print "enable the VIP: $vip on the new master: $new_master_host \n ";
      &start_vip();
      &start_arp();
############################################################################### 


###########################################################################  
sub start_vip() {  
    `ssh $new_master_ssh_user\@$new_master_host \" $ssh_start_vip \"`;  
}  
sub stop_vip() {  
    `ssh $orig_master_ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;  
}  

sub start_arp() {
    `ssh $new_master_ssh_user\@$new_master_host \" $ssh_bcast_arp \"`;
}
########################################################################### 

# vim master_ip_failover.3306

###########################################################################
my $vip = '10.3.9.26/21';  # Virtual IP 
my $key = "1"; 
my $ssh_start_vip = "/sbin/ifconfig ens192:$key $vip"; 
my $ssh_stop_vip = "/sbin/ifconfig ens192:$key down"; 
my $ssh_bcast_arp = "/sbin/arping -I ens192 -c 3 -A 10.3.9.26";
########################################################################### 


###########################################################################  
   print "Enabling the VIP - $vip on the new master - $new_master_host \n"; 
            &start_vip(); 
            &start_arp();
########################################################################### 


# A simple system call that enable the VIP on the new master 
###########################################################################  
sub start_vip() { 
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`; 
} 
# A simple system call that disable the VIP on the old_master 
sub stop_vip() { 
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`; 
} 
sub start_arp() {
    `ssh $ssh_user\@$new_master_host \" $ssh_bcast_arp \"`;
}    
###########################################################################

```


# 问题二：