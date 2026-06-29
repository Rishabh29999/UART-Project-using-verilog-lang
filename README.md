# UART-Project-using-verilog-lang
UART project using verilog by looking into FSM and Baudrate , input and outputs.
verilog code
// Code your testbench here
// or browse Examples
// Code 1: UART Transmitter Module with Parameter
module uart_tx #(
    parameter CLK_PER_BIT = 87 // Here we declared module & parameter.
                               // If Freq = 50MHz & Baud Rate = 115200:
                               // 50,000,000 / 115,200 ≈ 434.
                               // We use 87 to make the simulation wave shorter.
)(
    input        clk,          // sys. clk.
    input        rst_n,        // Asynch. reset.
    input        tx_start,     // kick off transmission.
    input [7:0]  tx_data,      // 8-bit data to send.
    output reg   tx_serial,    // Serial o/p line. 1-bit wire where data drops sequentially.
    output reg   tx_done       // High for 1 clk cycle when finished.
);

    // State machine using Parameters
    localparam STATE_IDLE  = 2'b00;
    localparam STATE_START = 2'b01;
    localparam STATE_DATA  = 2'b10;
    localparam STATE_STOP  = 2'b11;

    reg [1:0]  state, next_state;   // Just index also.
    reg [15:0] clk_count;           // Counter to measure bit duration.
    reg [2:0]  bit_index;           // Track which 8 bits are sending.
    reg [7:0]  data_buffer;         // Store data, doesn't change mid-transmission.
                                    // If we want, we can also name as data_exclude_buffer.

    // State Register
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state <= STATE_IDLE;
        end else begin
            state <= next_state;
        end
    end

    // Next State Logic
    always @(*) begin
        next_state = state;
        case (state)
            STATE_IDLE: begin
                if (tx_start) 
                    next_state = STATE_START;
            end
            
            STATE_START: begin
                if (clk_count == (CLK_PER_BIT - 1)) 
                    next_state = STATE_DATA;
            end
            
            STATE_DATA: begin
                if ((clk_count == (CLK_PER_BIT - 1)) && (bit_index == 7)) 
                    next_state = STATE_STOP;
            end
            
            STATE_STOP: begin
                if (clk_count == (CLK_PER_BIT - 1)) 
                    next_state = STATE_IDLE;
            end
            default: next_state = STATE_IDLE;
        endcase
    end

    // Datapath Logic (Counters, Buffers & Serial O/P)
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            clk_count   <= 0;
            bit_index   <= 0;
            data_buffer <= 0;
            tx_serial   <= 1'b1; // Idle state is High.
            tx_done     <= 1'b0;
        end else begin
            tx_done <= 1'b0;     // Default assignment.
            
            case (state)
                STATE_IDLE: begin
                    clk_count <= 0;
                    bit_index <= 0;
                    tx_serial <= 1'b1;
                    if (tx_start) begin
                        data_buffer <= tx_data;
                    end
                end
                
                STATE_START: begin
                    tx_serial <= 1'b0; // Here, 0 is Start bit. Start state mein '0' se shuru hoga.
                    
                    // Ek bit ko kitni der tak screen/line pr rakhna hain. 
                    // Jab tk tx_counter ki value CLK_PER_BIT - 1 tk nhi pahunchti, 
                    // tb tk hum counter ko +1 (increment) karenge.
                    if (clk_count < (CLK_PER_BIT - 1)) begin
                        clk_count <= clk_count + 1;
                    end else begin
                        clk_count <= 0; // Yahan Clk ko 0 kar diya hai taaki agle sire se naya count shuru ho.
                    end
                end
                
                STATE_DATA: begin
                    // Yahan hum ek-ek bit uthate hain & tx_serial line pr dal dete hain. (Same as above counter logic)
                    tx_serial <= data_buffer[bit_index]; // Send bit, LSB First (Least Significant Bit).
                    
                    if (clk_count < (CLK_PER_BIT - 1)) begin
                        clk_count <= clk_count + 1;
                    end else begin
                        clk_count <= 0;
                        
                        // Ye check kar rha hai ki humne saari bit bhej di? If no, toh ye (?) kategi (increment).
                        if (bit_index < 7) begin
                            bit_index <= bit_index + 1;
                        end else begin
                            bit_index <= 0;
                        end
                    end
                end
                
                STATE_STOP: begin
                    tx_serial <= 1'b1; // Pull line High for stop bit. (Same as above counter logic).
                    
                    if (clk_count < (CLK_PER_BIT - 1)) begin
                        clk_count <= clk_count + 1;
                    end else begin
                        tx_done   <= 1'b1;
                        clk_count <= 0; // Wapas kata diya, taaki naya data aaye toh shuru se shuru ho!
                    end
                end
            endcase
        end


      Test Bench

      // Code your design here
`timescale 1ns/1ps // 1ns precision & accuracy 1ps tak rahegi.

module uart_tx_tb;

    // Inputs (Regs)
    reg clk;
    reg rst_n;
    reg tx_start;
    reg [7:0] tx_data;

    // Outputs (Wires)
    wire tx_serial;
    wire tx_done;

    // Instantiate UUT (Unit Under Test) Override Parameter CLK_PER_BIT = 5
    // Note: A=10 (1010), B=11, C=12 etc. 
    uart_tx #(.CLK_PER_BIT(5)) uut (
        .clk(clk),
        .rst_n(rst_n),
        .tx_start(tx_start),
        .tx_data(tx_data),
        .tx_serial(tx_serial),
        .tx_done(tx_done)
    );

    // Clock Generation
    // Har 10ns ke baad clock flip hogi (0 se 1, ya 1 se 0).
    // 1 Complete cycle hone me = 20ns lagega.
    always #10 clk = ~clk;

    initial begin
        // Required for EDA Playground to generate VCD file for EPWave
        $dumpfile("dump.vcd");
        $dumpvars(0, uart_tx_tb);

        // Initialize Inputs
        clk      = 0;
        rst_n    = 0;
        tx_start = 0;
        tx_data  = 8'h00;

        // Apply Reset
        #40;
        rst_n = 1; // Simulation hote hi sys ko 40ns tk rst me rakhenge taaki saari internal registers clr ho.
                   // Uske baad rst ko 1 kar denge taaki normal operation shuru ho jaaye.

        // Send 1st Character: 8'hA5
        #20;
        tx_data  = 8'hA5; // Sabse pehle vo data transmit karenge jo krna hai. A5 -> 8'b10100101
        tx_start = 1;     // Ye Trigger ki tarah hai.
        
        #20;
        tx_start = 0;     // Hum wapas zero kar dete hain, ye Push-button ki tarah kaam karta hai.

        // Wait until transmission complete
        @(posedge tx_done);
        #40;

        // Send 2nd Character: 8'h3C
        tx_data  = 8'h3C;
        tx_start = 1;
        
        #20;
        tx_start = 0;

        // Wait until transmission complete
        @(posedge tx_done);
        #100;

        $display("Simulation Finished Successfully!");
        $finish;
    end

endmodule
    end

endmodule
