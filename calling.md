Recommended calling convention
The stack can be used to maintain function stack frames. Consider the following code:
:CODE_START
//Top of stack is 0x03FF
//Bottom bound of stack is 0x0000
//SP starts at 0x0400
//BP starts at 0x03FF
//RAM[SP] contains garbage!
//RAM[BP] contains garbage!
PUSH 1 //local variable a
PUSH 2 //local variable b
PUSH 0 //local variable c
//these are all local variables to the code.
//a is located at SP+2, BP-0, 0x03FF
//b is located at SP+1, BP-1, 0x03FE
//c is located at SP+0, BP-2, 0x03FD
//we now do some stuff, specifcally, c = a+b
LOAD AX BP  0 //AX now contains a (1)
LOAD BX BP -1 //BX now contains b (2)
ADD  CX AX BX //CX now contains a+b (3)
STOR CX BP -2 //c now contains CX (3)
//now we want to call MUL(c, 10) and assign the result to a
//if we want to preserve any registers, we should push them now.
PUSH 10
LOAD AX BP -2
PUSH AX
//10 is located at SP+1
//c is located at SP+0
CALL MUL //resolves to CJMP 7 X1|2 X2 VV, probably CJMP 7 6 0 MUL, PC+1 is pushed to stack.
//we have returned, convention specifies that the return value or address is in DX
POP ZZ
POP ZZ //clean the stack.
//we are now in the state we were before the call, except that General and Flags Registers are undefined,
//and DX contains the return.
STOR DX BP 0
//a = return value
//now halt:
JMP 0 //unconditional relative jump to this location.
:MUL //arguments(x, b) returns r
PUSH BP //push the old base pointer onto the stack
SET BP SP //BP = SP
SUB SP SP 1 //SP -= 1 //reserving space for local variable r.
            //we could also just PUSH ZZ
//argument b is located at BP+3
//argument a is located at BP+2
//return address is located BP+1
//old BP is located at BP
//local variable r is located at BP-1
LOAD AX BP +3  //AX = a
LOAD BX BP +2  //BX = b
ADD AX AX BX   //AX = AX+BX (a+b)
STORE AX BP -1 //r = AX (a+b)
LOAD DX BP -1  //DX = r //return value
ADD SP SP 1    //free local variables
POP BP         //restore old BP
RET            //resolves to POP CX, JMP CX
