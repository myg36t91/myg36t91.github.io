---
title: Home
nav_order: 1
---

# Home

```verilog
`timescale 1ns/1ps

model home (
   input        clk,
   output reg   dout
);

always @ (podedge clk) begin
   dout <= Welcome Home Page ;
end

endmodel
```

```markdown
### Hi there.

##### I’m currently studying master degree in NTUST EE.
Lab research topics include next-generation networks, information security and artificial intelligence, etc. <br>
Other fields and skills are learned based on personal interests.

- Languages
  - Chinese : Native
  - English : TOEIC 545
  - Japanese : JLPT N2
  
- Skills
  - Verilog HDL
  - Python
  - Field Programmable Gate Array
  - Git

- Project
  - 5G O-RAN Security Countermeasures
  - Tactile Internet 5G/B5G Design - Security Aspects
  - Deploy Information Security System on the AI Chip

- Paper
  - C.T. Shen; Y.Y. Xiao; Y.W. Ma; J.L. Chen; Cheng-Mou Chiang; S.J. Chen; Y. C. Pan, "Security Threat Analysis and Treatment Strategy for O-RAN" *2022 24th International Conference on Advanced Communication Technology (ICACT)*
 
Currently I responsible for module FPGA IP develop in company. <br>
On the other hand, I am also in charge of AI Box Security project of the ITRI now.
```

