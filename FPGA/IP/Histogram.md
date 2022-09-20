---
title: Histogram
parent: IP
grand_parent: FPGA
has_children: false
nav_order: 1
---

# Histogram

##### RAM_Generator
```verilog
`timescale 1ns / 1ps

module RAM_Generator
	#(
		parameter C_DATA_WIDTH   = 8,
		parameter C_DEPTH        = 16,
		parameter C_CE_IN        = 0
	)
	(
		input  clk_i,
		input  clk_en_i,
		input  rstn_i,
		input  write_en_i,
		
		input  [$clog2(C_DEPTH) - 1 : 0] write_address_i,
		input  [C_DATA_WIDTH - 1 : 0]    write_data_i,

		input  [$clog2(C_DEPTH) - 1 : 0] read_address_i,
		output [C_DATA_WIDTH - 1 : 0]    read_data_o
	);

	reg [C_DATA_WIDTH - 1 : 0] read_data_r;
	reg [C_DATA_WIDTH - 1 : 0] buffer [C_DEPTH - 1 : 0];

	wire clk_en = C_CE_IN ? clk_en_i : 1;

	assign read_data_o = read_data_r;
	
    integer i;
    initial begin
        for (i = 0; i < C_DEPTH; i = i + 1) begin
            buffer[i] <= 0; 
        end
        read_data_r <= 0;
    end
    
    always @ (posedge clk_i) begin
        if (clk_en) begin
            if (write_en_i) begin
                buffer[write_address_i] <= write_data_i;
            end
            read_data_r <= buffer[read_address_i];
        end
    end

endmodule
```
##### Histogram RTL
```verilog
`timescale 1ns / 1ps

module Histogram
    #(
        parameter C_QUANTITY = IMG_HEIGHT * IMG_WIDTH,
        parameter C_DATA_WIDTH = $clog2(C_QUANTITY) + 1
        
    )
    (
        input clk_i,
        input rstn_i, 
        input valid_i,
        input write_en_i,
        input last_i,
        input [C_DATA_WIDTH - 1 : 0] data_i,
       
        output last_o,
        output valid_o,
        output [$clog2(C_QUANTITY) : 0] data_o
    );
    
    localparam IMG_HEIGHT = 32;
    localparam IMG_WIDTH  = 32;
    localparam PIXEL      = 256;
    
    // Parameter declaration
    reg valid_i_r;
    reg valid_o;
    reg last_o;                                      
    reg [C_DATA_WIDTH - 1 : 0] data_i_r;
                                     

    wire [$clog2(C_QUANTITY) : 0] write_address;            // statistics pixel as write address
    wire [$clog2(C_QUANTITY) : 0] read_address;          
    
    wire inc_rst;                                           // increment_reset
    wire inc_en;                                            // increment_enable
    
    reg [C_DATA_WIDTH - 1  : 0] out_pixel_count;
    reg [C_DATA_WIDTH - 1  : 0] out_pixel_count_r;
                                   
    wire [C_DATA_WIDTH - 1 : 0] inc_data;                   // data = count + count_feedback
    wire [C_DATA_WIDTH - 1 : 0] inc_count_fb;                // increment_count_feedback
    reg  [C_DATA_WIDTH - 1 : 0] inc_count;                  // increment_count
    reg  statistics_flag;                                   // 1: statistics, 0: output
    
    // Delay data_i by clock cycle
    always @ (posedge clk_i) begin
        if (!rstn_i) begin
            valid_i_r  <= 1'b0;
            data_i_r   <= {C_DATA_WIDTH{1'b0}};
        end
        else begin
            valid_i_r  <= valid_i;
            data_i_r   <= data_i;
        end
    end   

    // Statistics increment counter
    always @ (posedge clk_i) begin
        if (inc_rst)
            inc_count <= {{$clog2(C_QUANTITY)-1{1'b0}}, 1'b1};
        else if (inc_en)
            inc_count <= inc_count + {{$clog2(C_QUANTITY)-1{1'b0}}, 1'b1};
        else
            inc_count <= inc_count;
    end
    
    // Switch statistics and output flag
    always @ (posedge clk_i) begin
        if (!rstn_i) begin
            statistics_flag <= 1'b1;
        end
        else begin
            if (last_i & valid_i_r) begin
                statistics_flag <= 1'b0;
            end
            else begin
                statistics_flag <= statistics_flag;
            end
        end
    end
    
    // Output data_o
    always @ (posedge clk_i) begin
        if (statistics_flag) begin
            out_pixel_count <= 1'b0;
            out_pixel_count_r <= 1'b0;  
        end
        else begin
            if (!statistics_flag & out_pixel_count < PIXEL) begin
                out_pixel_count <= out_pixel_count + 1'b1;
                out_pixel_count_r <= out_pixel_count;
                last_o <= 1'b0;
                valid_o <= 1'b1;
            end
            else if (out_pixel_count_r == (PIXEL - 1) & valid_o) begin
                last_o <= 1'b1;
                valid_o <= 1'b0;
            end
            else begin
                statistics_flag <= 1'b1;
                last_o <= 1'b0;
                valid_o <= 1'b0;
            end
        end
    end   

    assign inc_rst = ((data_i_r != data_i) | ((valid_i == 1'b1) & (valid_i_r == 1'b0))) ? 1'b1 :1'b0;
    assign inc_en = ((data_i_r == data_i) & (valid_i == 1'b1) & (valid_i_r == 1'b1)) ? 1'b1 : 1'b0;
    assign inc_data = (statistics_flag == 1'b1) ? inc_count + inc_count_fb : 1'b0;
    assign write_en_i = (!inc_en & valid_i_r == 1'b1) ? 1'b1 : 1'b0;
    assign read_address = (statistics_flag == 1'b1) ? data_i : out_pixel_count;
    assign write_address = data_i_r;
    assign data_o = (statistics_flag == 1'b0 & valid_o) ? inc_count_fb : data_o;

    RAM_Generator
    #(
        .C_DATA_WIDTH(C_DATA_WIDTH),
        .C_DEPTH(256)
    )
    RAM0
    (
        .clk_i(clk_i),
        .rstn_i(rstn_i),
        .write_en_i(write_en_i),
        .write_address_i(write_address),
        .read_address_i(read_address),
        .write_data_i(inc_data),
        .read_data_o(inc_count_fb)
    );
    
endmodule
```
##### Histogram Testbench
```verilog
`timescale 1ns / 1ps

module Histogram_tb();   

    localparam IMG_HEIGHT = 32;
    localparam IMG_WIDTH = 32;
    
    parameter C_QUANTITY = IMG_HEIGHT * IMG_WIDTH;
    parameter C_DATA_WIDTH = $clog2(C_QUANTITY) + 1;
    
    reg clk_i;
    reg rstn_i;
    reg valid_i;
    reg write_en_i;
    reg last_i;
    reg [C_DATA_WIDTH - 1 : 0] data_i;
    
    wire last_o;
    wire valid_o;
    wire [$clog2(C_QUANTITY) : 0] data_o;
    
    localparam DATA_RANGE = 255;
    localparam NUM_TEST_DATA = 1024;
    
    reg _vld = 0;
    
    initial begin
        clk_i      = 0;
        rstn_i     = 0;
        valid_i    = 0;
        last_i     = 0;
    end
    
    initial begin
        # 2 
        forever # 5 clk_i = ~clk_i;
    end
    
    integer i;
    initial begin
        # 20 rstn_i = ~rstn_i;
        for (i = 0; i <= NUM_TEST_DATA; i = i + 1) begin
            @ (posedge clk_i) begin           
                _vld = $urandom_range(1, 0);
                valid_i <= _vld;
                if (_vld == 1) begin
                    data_i <= $urandom_range(DATA_RANGE); // 0 ~ DATA_RANGE
                   $display("The clk number: %d and data: %d.", i, data_i);
                    if (i < NUM_TEST_DATA) begin
                        last_i <= 0;
                    end
                    else if (i == NUM_TEST_DATA) begin
                        last_i <= 1;
                        valid_i <= 0;
                    end
                end
                else begin
                    last_i <= 0;
                    i = i - 1;
                end
            end
        end
    end
    
    Histogram Histogram_uut
    (
       .clk_i(clk_i),
       .rstn_i(rstn_i),
       .valid_i(valid_i),
       .write_en_i(),
       .last_i(last_i),
       .data_i(data_i),
       .last_o(last_o),
       .valid_o(valid_o),
       .data_o(data_o)       
    );
    
    // Histogram_tb.v Block and Non-block Test
    //    reg dff_sample;
    //    always @ (posedge clk_i) begin
    //        dff_sample <= valid_i;
    //    end
    
endmodule
```
