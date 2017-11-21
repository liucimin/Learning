#Ovs bond 

```c
bond_choose_output_slave
	choose_output_slave
		e = lookup_bond_entry(bond, flow, vlan);
```





静态汇聚原理

​		动态汇聚和静态汇聚原理类似，只是动态汇聚中所有端口都是通过协议确定，而不是像静态汇聚通过协议在指定端口中确定汇聚相关端口。