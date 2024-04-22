# ZRX Pathways Experiment 2 (E-2): Optimistic Funding for 0x Improvement Proposals

> [!NOTE]
> refer to [pZEIP procedure](./pZEIPs/README.md) for instructions on how to submit a pZEIP.›

## Purpose
ZRX Pathways is a community-based initiative that aims to make the protocol more performant, innovative, and sustainable via the implementation of enabling structures and actionable pathways, providing a better experience for core protocol contributors.

0x Improvement Proposals (ZEIPs) describe standards and protocol specifications for the 0x protocol.

Core protocol contributors working on ZEIPs may need funding to resource their efforts. 

**This experiment is designed to improve the contributor experience by implementing optimistic funding for grants to ZRX Pathways participants working on preliminary ZEIPs (pZEIPs).**

## Correlation to ZRX Pathways Experiment 1 (E-1)
This experiment improves upon Experiment 1 (E-1), where an accelerated process was designed to improve the contributor experience by removing friction and administrative overhead, enabling developers to focus on the job to be done. 

### E-1 Hypothesis
> We believe that a bespoke grant application and process will result in faster funding. We will know we have succeeded when we see an increase in contributor applications, which is a signal that more contributors are comfortable with the process.

### E-1 Outcome
The experiment was successful insofar as it proved the hypothesis when multiple developers/teams initiated pZEIPs. 

### E-1 Learnings 
E-1 had the following goals:
>1. Frontload technical rigor associated with ZEIPs to inform and accelerate the development process
>2. Reduce the administrative burden of managing information tangential to the development process
>3. Mitigate risks associated with accelerated decision-making for community-based funding
>4. Reduce the time between application submittal and funds disbursal

Regarding these goals, the following learnings were achieved:
1. The requirement for technical specifications was an improvement over the status quo for general treasury proposals and enabled more structured and informed decision-making.
2. The reduction of administrative burden in managing process-type (i.e., non-technical) information tangential to the development process was positive in that it abstracted these details for grant seekers, removing friction. 
3. The risk mitigation measures were overall appropriate and beneficial. However, milestone funding can inadvertently disadvantage and/or create challenges for contributors when they deliver on their plan, but future milestone funding is not approved in a subsequent treasury vote. 
4. The shortened timeline was an improvement over the status quo (for general treasury proposals), and there is no clear evidence that a longer timeline would provide improved outcomes for these proposals. Note that this learning is specific to the conditions of this experiment, which included multiple risk mitigation factors, and does not offer proof that the treasury grant timeline should also be reduced. 

### E-1 Conclusions
The improvements were consequential but still insufficient to achieve the overall goal of enabling a better experience for core protocol contributors. A qualified contributor meeting the defined requirements of the initiative was unable to obtain a milestone grant, resulting in a suboptimal experience for them and delays in development and innovation for the protocol. 

The current quorum requirement for treasury votes, combined with voter behavior, prevents qualified contributors from obtaining grants and discourages contributors from participating. This insufficiency provides the impetus for Experiment 2 (E-2) (described below).

## Experiment Design
Similar to E-1, E-2 is designed to produce learnings and improvements related to the following desired outcome. It incorporates a user story + hypothesis framing:

### Desired Outcome
> How might we make it easier for developers to contribute to the core protocol contracts?

### User Story
> As a [role], I want [goal/desire], so that [benefit].

As a developer, I want to minimize the time and effort associated with grant processes, so that I can focus on the functional job to be done.

### Hypothesis
> We believe [this capability] will result in [this outcome]. We will know we have succeeded when [measurable signal].

We believe that implementing optimistic funding for protocol contributor grants will result in more predictable, reliable, and consistent funding for qualified contributors. We will know we have succeeded when we see an increase in grants awarded to protocol contributors.

## Experiment Components
This experiment is designed for protocol contributors who are seeking funding from the treasury and has two goals:

> 1. Reduce the friction associated with funding for pZEIP contributors
> 2. Increase the predictability, reliability, and consistency that qualified contributors will receive grants

This experiment has three components:

> 1. **Process**: 2-week cycle with concurrent forum discussion and Snapshot voting
> 2. **Voting**: Snapshot subspace with ZRX voting power strategy and optimistic quorum
> 3. **Execution/Disbursement**: Tellor Zodiac Module (oracle) and Gnosis Safe

In combination, these three components enable optimistic approval and execution of protocol contributor grants, providing that certain criteria are met (see Considerations, below). 

### Process
Similar to E-1, the forum discussion and Snapshot vote will run concurrently, but the duration will be extended to 14 days to reflect that the onchain vote and associated queue/delay is eliminated. 

### Voting
Voting will take place in Snapshot. A dedicated subspace connected to the main space will be implemented for pZEIP funding votes. Voting will use the same ZRX voting power strategy used for all 0x Snapshot votes, but will use the optimistic quorum strategy to determine the vote outcome. The quorum will be set to 7M ZRX. 

Specifically, the outcome will be determined as follows:

1. If the votes AGAINST votes do not exceed the quorum (7M ZRX), the vote will pass. 
2. If the votes AGAINST equal or exceed the quorum (7M ZRX), the vote will fail.

### Execution/Disbursement
Grants will be disbursed using the Tellor Zodiac module and a Gnosis Safe. The Tellor Zodiac module is a Tellor implementation of the Gnosis Reality Module, and it uses the Tellor oracle to enable onchain execution from a Gnosis Safe based on Snapshot proposal results. 

## Considerations
While the optimistic funding process streamlines and, by default, approves funding, it uses limited, fenced funds to balance the associated risk. Additionally, this experiment incorporates the risk mitigation measures defined in E-1, i.e., grant requestors must have a compliant and merged PR in the pZEIP repo, and grants are subject to a maximum $25k per milestone payment, or audit cost if the payment is for an audit. 
