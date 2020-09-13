# Paxos
基本定义

算法中的参与者主要分为三个角色，同时每个参与者又可兼领多个角色:  
```
⑴、proposer 提出提案，提案信息包括提案编号和提议的value;  
⑵、acceptor 收到提案后可以接受(accept)提案;  
⑶、learner 只能"学习"被批准的提案;
```
算法保重一致性的基本语义:
```
⑴、决议(value)只有在被proposers提出后才能被批准(未经批准的决议称为"提案(proposal)");  
⑵、在一次Paxos算法的执行实例中，只批准(chosen)一个value;  
⑶、learners只能获得被批准(chosen)的value;
``` 
四个约束:
```
⑴、P1: 一个acceptor必须接受(accept)第一次收到的提案;  
⑵、P2a:一旦一个具有value v的提案被批准(chosen)，那么之后任何acceptor 再次接受(accept)的提案必须具有value v;  
⑶、P2b:一旦一个具有value v的提案被批准(chosen)，那么之后任何 proposer 提出的提案必须具有value v;  
⑷、P2c:如果一个编号为n的提案具有value v，那么存在一个多数派，要么他们中所有人都没有接受(accept)编号小于n的任何提案，要么他们已经接受(accpet)的所有编号小于n的提案中编号最大的那个提案具有value v;  
``` 

1. prepare阶段：
2. accept批准阶段： 
* Classic Paxos的几个阶段：
```
Phase1a：Leader提交proposal到Acceptor

Phase2b：Acceptor回应已经参与投票的最大Proposer编号和选择的Value

Phase2a：Leader收集Acceptor的返回值

Phase2a.1：如果Acceptor无返回值，则自由决定一个
Phase2a.2： 如果有返回值，则选择Proposer编号最大的一个
Phase2b：Acceptor把表决结果发送到Learner
```
# Fast Paxos
```
Phase1a：Leader提交proposal到Acceptor
Phase1b：Acceptor回应已经参与投票的最大Proposer编号和选择的Value
Phase2a：Leader收集Acceptor的返回值
Phase2a.1：如果Acceptor无返回值，则发送一个Any消息给Acceptor，之后Acceptor便等待Proposer提交Value
Phase2a.2：如果有返回值，则根据规则选取一个
Phase2b：Acceptor把表决结果发送到Learner（包括Leader）
```
冲突  
冲突Recovery  
*基于协调者的Recovery*  
基于非协调的Recovery  
* Fast Paxos Progress
```
Phase1a：Leader提交proposal到Acceptor
Phase1b：Acceptor回应已经参与投票的最大Proposer编号和选择的Value
Phase2a：Leader收集Acceptor的返回值
Phase2a.1：如果Acceptor无返回值，则发送一个Any消息给Acceptor，之后Acceptor便等待Proposer提交Value
Phase2a.2：如果有返回值
      2.1 如果仅存在一个Value，则作为结果提交
      2.2 如果存在多个Value，则根据O4(v)选取符合条件的一个
      2.3 如果存在多个结果并且没有符合O4(v)的Value，则自由决定一个
Phase2b：Acceptor把表决结果发送到Learner（包括Leader）
```