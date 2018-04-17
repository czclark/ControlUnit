# ControlUnit
module ControlUnit(instruction, status, reset, clock, control_word/*, literal*/);
	input [31:0] instruction;
							  // registerd   , instant
	input [4:0] status; // {V, C, N, Z}, Z
	input reset, clock;
	output [CW_BITS:0] control_word;
	//output [63:0] literal;
	
	parameter CW_BITS = 96;
	
	wire [10:0] opcode;
	assign opcode = instruction[31:21];
	
	// partial control words
	wire [32:0] branch_cw, other_cw;
	wire [32:0] D_format_cw, I_arithmetic_cw, I_logic_cw, IW_cw, R_ALU_cw;
	wire [32:0] B_cw, B_cond_cw, BL_cw, CBZ_cw, BR_cw;
	
	// state logic
	wire NS;
	reg state;
	always @(posedge clock or posedge reset) begin
		if(reset)
			state <= 1'b0;
		else
			state <= NS;
	end
	
	// partial control unit decoders
	D_decoder dec0_000 (instruction, D_format_cw);
	I_arithmetic_decoder dec0_010 (instruction, I_arithmetic_cw);
	I_logic_decoder dec0_100 (instruction, I_logic_cw);
	IW_decoder dec0_101 (instruction, state, IW_cw);
	R_ALU_decoder dec0_110 (instruction, R_ALU_cw);
	B_decoder dec1_000 (instruction, B_cw);
	B_cond_decoder dec1_010 (instruction, status[4:1], B_cond_cw);
	BL_decoder dec1_100 (instruction, BL_cw);
	CBZ_decoder dec1_101 (instruction, status[0], CBZ_cw);
	BR_decoder dec1_110 (instruction, BR_cw);
	
	// 2:1 mux to select between branch instructions and all others
	assign control_word = opcode[5] ? branch_cw : other_cw;
	
	// 8:1 mux to select between branch insturctions
	Mux8to1Nbit branch_mux (branch_cw, opcode[10:8], 
		B_cw, 0, B_cond_cw, 0, BL_cw, CBZ_cw, BR_cw, 0);
	defparam other_mux.N = CW_BITS+1;	
	
	// 8:1 mux to select between all other insturctions
	Mux8to1Nbit other_mux (other_cw, opcode[4:2], 
		D_format_cw, 0, I_arithmetic_cw, 0, I_logic_cw, IW_cw, R_ALU_cw, 0);
	defparam branch_mux.N = CW_BITS+1;
	
endmodule

module D_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS, EN_B;
	wire C0;
	
	assign C0 = 1'b0;
	assign DA = I[4:0];
	assign SA = I[9:5];
	assign SB = 5'b0;
	assign K = I[31] ? {55'b0, I[20:12]} : {I[20],3'b0,I[19],3'b0,I[18],3'b0,I[17],3'b0,I[16],3'b0,I[15],
	                                        3'b0,I[14],3'b0,I[13],3'b0,I[12],3'b0};
	assign PS = 2'b01;
	assign FS = 5'b01000; 
	assign SL = 1'b0;
	assign Bsel = 1'b1;
	assign PCsel = 1'b0;
	assign EN_PC = 1'b0;
	assign NS = 1'b0;
	
	assign EN_ALU = 1'b0;
   assign EN_RAM = I[22] ? 1'b1 : 1'b0;
	assign EN_B = I[22] ? 1'b0 : 1'b1;
   assign WM = I[22] ? 1'b0 : 1'b1;
   assign WR = I[22] ? 1'b1 : 1'b0;	
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS, K};
		
endmodule

module I_arithmetic_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
// Declarations
	wire C0;
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS, EN_B;
	
// Control signal def.
	assign K = {52'b0, I[21:10]};
	assign DA = I[4:0];
	assign SA = I[9:5];
	assign SB = 5'b00000;
   assign WM = 1'b0;
	assign PS = 2'b01;
	assign EN_ALU = 1'b1;
	assign EN_RAM = 1'b0;
	assign EN_B = 1'b0;
	assign WR = 1'b1;
	assign Bsel = 1'b1;
	assign NS = 1'b0;
	assign EN_PC = 1'b0;
	assign PCsel = 1'b0;
	
	assign C0 = I[31] & I[30] & ~I[26] & I[24] & ~I[23] & ~I[22];
	assign FS = {4'b0100,I[30]};
	assign SL = I[29];
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS, K};
	
endmodule

module I_logic_decoder (I, control_word);  // Come back to 
	input [31:0]I;
	output [96:0] control_word;
	
// Declarations
	wire C0;
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS, EN_B;
	
// Control signal def.
	assign K = {52'b0, I[21:10]};
	assign DA = I[4:0];
	assign SA = I[9:5];
	assign SB = 5'b0;
   assign WM = 1'b0;
	assign PS = 2'b01;
	assign EN_ALU = 1'b1;
	assign EN_RAM = 1'b0;
	assign WR = 1'b1;
	assign Bsel = 1'b1;
	assign NS = 1'b0;
	assign EN_PC = 1'b0;
	assign PCsel = 1'b0;
	assign EN_B = 1'b0;
	
	assign C0 = I[31] & I[30] & ~I[26] & I[24] & ~I[23] & ~I[22];
	assign FS = ((I[30] & I[29]) | ~(I[30] & I[29])) ? 5'b00000 : (I[29] ? 5'b00100 : 5'b01100);
	assign SL = I[30] & I[29];
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS, K};
	
endmodule

module IW_decoder (I, state, control_word);
	input [31:0]I;
	input state;
	output [96:0] control_word;
	
	// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;
	
	// easy stuff first
	assign DA = I[4:0];
	assign SB = 5'b0;
	assign Bsel = 1'b1;
	assign PCsel = 1'b0;
	assign SL = 1'b0;
	assign EN_ALU = 1'b1;
	assign EN_RAM = 1'b0;
	assign EN_PC = 1'b0;
	assign WM = 1'b0;
	assign EN_B = 1'b0;
	assign WR = 1'b1;
	assign C0 = 1'b0;
	
	// hard ones
	wire I29_Snot;
	assign I29_Snot = I[29] & ~state;
	assign SA = I[29] ? I[4:0] : 5'd31;
	assign K = (I29_Snot) ? 64'hFFFFFFFFFFFF0000 : {48'b0, I[20:5]};
	assign PS = {1'b0, ~I29_Snot};
	assign FS = {2'b00, ~I29_Snot, 2'b00};
	assign NS = I29_Snot;
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS, K};
	
endmodule    //Actual CW: SA, SB, DA, RegWrite, MemWrite, FS, C0, EN_Mem, EN_ALU, Asel,Bsel,PS, EN_PC, EN_B, SL, NS, K

module R_ALU_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
	// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS, EN_B;
	
	// ControlWord translations
	assign DA = I[4:0];
	assign SA = I[9:5];
	assign SB = I[20:16];
	assign EN_ALU = 1'b1;
	assign EN_RAM = 1'b0;
	assign EN_B = 1'b0; 
	assign EN_PC = 1'b0;
	assign PS = 2'b01;
	assign WM = 1'b0;
	assign WR = 1'b1;
	assign PCsel = 1'b0;
	assign NS = 1'b0;
	
	assign K = I[15:10];
	assign C0 = I[31] & I[30] & ~I[26] & I[24] & ~I[23] & ~I[22];
	assign FS = I[22] ? (I[21] ? 5'b10000 : 5'b10100) : (I[24] ? (I[30] ? 5'b01001 : 5'b01000) : 
				   ((I[30] & I[29]) | ~(I[30] & I[29]) ? 5'b00000 : (I[29] ? 5'b00100 : 5'b01100)));
				  
	assign Bsel = I[22];
	assign SL = I[29] & (I[30] | I[24]);
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};
	
endmodule

module B_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
		// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;

	assign DA = 5'b0;
	assign SA = 5'b0;
	assign SB = 5'b0;
   assign Bsel = 1'b0;
	assign WM = 1'b0;
	assign WR = 1'b0;
	assign SL = 1'b0;
	assign EN_ALU = 1'b0;
	assign EN_RAM = 1'b0;
	assign C0 = 1'b0;
	assign EN_B = 1'b0;
	assign NS = 1'b0;
	
	assign FS = 5'b0;
	assign K[63:26] = {38{I[25]}};
	assign K[25:0] = I[25:0];
	assign EN_PC = 1'b0;
	assign PCsel = 1'b1;
	assign PS = 2'b10;
	
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};
endmodule

module B_cond_decoder (I, status, control_word);
	input [31:0]I;
	input [3:0]status;
	output [96:0] control_word;
	
			// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;

	assign DA = 5'b0;
	assign SA = 5'b0;
	assign SB = 5'b0;
   assign Bsel = 1'b0;
	assign WM = 1'b0;
	assign WR = 1'b0;
	assign SL = 1'b0;
	assign EN_ALU = 1'b0;
	assign EN_RAM = 1'b0;
	assign C0 = 1'b0;
	assign EN_B = 1'b0;
	assign NS = 1'b0;
	
	assign FS = 5'b0;
	assign K = {{45{I[25]}},I[25:5]};
	assign EN_PC = 1'b0;
	assign PCsel = 1'b1;
	
	cond_encoder ce (PS, I[3:0], status);

	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};
endmodule

module BL_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
			// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;

	assign SA = 5'b0;
	assign SB = 5'b0;
   assign Bsel = 1'b0;
	assign WM = 1'b0;
	assign SL = 1'b0;
	assign EN_ALU = 1'b0;
	assign EN_RAM = 1'b0;
	assign C0 = 1'b0;
	assign EN_B = 1'b0;
	assign NS = 1'b0;
	
	assign DA = 5'b11110;
	assign WR = 1'b1;
	assign FS = 5'b0;
	assign K = {{38{I[25]}},I[25:0]};
	assign EN_PC = 1'b1;
	assign PCsel = 1'b1;
	assign PS = 2'b10;
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};
endmodule

module CBZ_decoder (I, status,control_word);
	input [31:0]I;
	input status;
	output [96:0] control_word;
				// wire for all control signals
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;


   assign DA = 5'b0;
	assign Bsel = 1'b0;
	assign WM = 1'b0;
	assign WR = 1'b1;
	assign EN_ALU = 1'b0;
	assign EN_RAM = 1'b0;
	assign C0 = 1'b0;
	assign EN_B = 1'b0;
	assign NS = 1'b0;
	
	assign SL = 1'b0;
	assign SA = 5'b11111;
	assign FS = 5'b01000;
	assign SB = I[4:0];
	assign K = {{45{I[25]}},I[25:5]};
	assign EN_PC = 1'b0;
	assign PCsel = 1'b1;
	assign PS = (I[24] ? (status == 1'b0) : (status == 1'b1)) ? 2'b11 : 2'b01;
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};
endmodule

module BR_decoder (I, control_word);
	input [31:0]I;
	output [96:0] control_word;
	
	wire [4:0] DA, SA, SB;
	wire [63:0] K;
	wire [1:0] PS;
	wire [4:0] FS;
	wire PCsel, Bsel, SL, EN_ALU, EN_RAM, EN_PC, WM, WR, NS;
	wire C0, EN_B;


   assign DA = 5'b0;
	assign Bsel = 1'b0;
	assign WM = 1'b0;
	assign WR = 1'b1;
	assign EN_ALU = 1'b0;
	assign EN_RAM = 1'b0;
	assign C0 = 1'b0;
	assign EN_B = 1'b0;
	assign NS = 1'b0;
	
	assign SL = 1'b0;
	assign SA = I[4:0];
	assign FS = 5'b00000;
	assign SB = 5'b00000;
	assign K = 64'b0;
	assign EN_PC = 1'b0;
	assign PCsel = 1'b0;
	assign PS = 2'b10;
	
	assign control_word = {SA, SB, DA, WR, WM, FS, C0, EN_RAM, EN_ALU, PCsel, Bsel, PS, EN_PC, EN_B, SL, NS,K};	
endmodule

module cond_encoder(PS,cond,status);
		input [3:0] cond;
		input [3:0] status;
		output reg [1:0] PS;
		// V C N Z
		always@(*) case(cond)
		
		4'b0000: PS = status[0] ? 2'b11 : 2'b01;
		
		4'b0001: PS = ~status[0] ? 2'b11 : 2'b01;
		
		4'b0010: PS = status[2] ? 2'b11 : 2'b01;
		
		4'b0011: PS = ~status[2] ? 2'b11 : 2'b01;
		
		4'b0100: PS = status[1] ? 2'b11 : 2'b01;
		
		4'b0101: PS = ~status[1] ? 2'b11 : 2'b01;
		
		4'b0110: PS = status[3] ? 2'b11 : 2'b01;
		
		4'b0111: PS = ~status[3] ? 2'b11 : 2'b01;
		
		4'b1000: PS = (~status[0] & status[2]) ? 2'b11 : 2'b01;
		
		4'b1001: PS = (status[0] & ~status[2]) ? 2'b11 : 2'b01;
		
		4'b1010: PS = ~(status[3]^status[1]) ? 2'b11 : 2'b01;
		
		4'b1011: PS = (status[3]^status[1]) ? 2'b11 : 2'b01;
		
		4'b1100: PS = (~status[0] & ~(status[3] & status[1])) ? 2'b11 : 2'b01;
		
		4'b1101: PS  = (status[0] | (status[3] & status[1])) ? 2'b11 : 2'b01;
		
		4'b1110: PS = 2'b11;
		
		4'b1111: PS = 2'b11;
		endcase
		
endmodule
