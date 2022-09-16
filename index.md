---
title: Home
nav_order: 1
---

# Home
```verilog
`timescale 1ns/1ps

model home (
   input    clk,
   input    rstn,
   output   dout
);

reg dout ;

always @ (podedge clk) begin
   if (!rstn) begin
      dout <= 0 ;
   end
   else
      dout <= 「Welcome Home Page.」 ;
   end
end

endmodel
```

