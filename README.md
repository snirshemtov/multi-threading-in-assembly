# multi-threading-in-assembly
https://www.cs.bgu.ac.il/~caspl192/Assignments/Assignment_3


Assignment 3: multi-threading in assembly
Assignment Description
Note the fixes in the assignment:
create initial configuration of the game in the following order :
first target, then drones
first x coordinate, they y coordinate, then angle
you may use malloc() and free() C standard library functions
scaling process of random number
suppose you generated integer random number x in unsigned range [0,MAXINT]
suppose you need to scale this number to your [0,100] board size to calculate some coordinate
then, just execute the following calculation: x_scaled = float point value of (x / MAXINT) * 100
when print game board, print float point numbers in "fixed format" (%f, or %lf, etc.) with 2 digits after the decimal point.

Your code will be written entirely in assembly language. No C code is allowed, and no usage of C standard library functions other than those specifically noted below. The only C standard library code allowed is the initialization code at "_start" which calls main( ), usage of printf and fprintf functions to print float point numbers and integers (and whatever else you need to print), sscanf in order to convert the command line arguments. Use of malloc( ) and free( ) is also allowed, to handle your dynamic memory allocation.
You are to write a simple user-mode co-routine ("multi-thread") simulation, using the co-routine mechanism code descibed and provided in class. The goal is further proficiency in assembly language, especially as related to floating point, and understanding stack manipulations, as needed to create a co-routine (equivalently thread) management scheme.

Program Goal
You will implement a multi-drones single-target game. Suppose a 100x100 game board. Suppose group of N drones which see the same target from different points of view and from different distance. Each drone tries to detect where is the target on the game board, in order to destroy it. Drones may destroy the target only if the target is in drone’s field-of-view, and if the target is no more than some maximal distance from the drone. When the current target is destroyed, some new target appears on the game board in some randomly chosen place. The first drone that destroys T targets is the winner of the game. Each drone has three-dimensional position on the game board: coordinate x, coordinate y, and direction (angle from x-axis). Drones move randomly chosen distance in randomly chosen angle from their current place. After each movement, a drone calls mayDestroy(…) function with its new position on the board. mayDestroy(…) function returns TRUE if the caller drone may destroy the target, otherwise returns FALSE. If the current target is destroyed, new target is created at random position on the game board. Note that drones do not know the coordinates of the target on the board game. On the example below, the blue drone does not see the target, the yellow drone sees the target, but the target is very far from it, and the pink drone both sees the target and close enough to it. The pink drone can destroy the target.
Please note: the target is just a point, not a circle as it appears in the figures. The painting of a circle is just for the convenience of illustration. Also note that drones are illustrated as airplanes but they do not have all the appropriate functionality of airplanes.

assign3/ass3_fig1.png

When a drone moves from its current position to a new position, it may happen that the distance move makes the drone cross the game field border. We treat drone motion as if on a torus. That is, when the next location is greater than the board width or height, or is negative in either axis (this requires checking 4 conditions), subtract or add the torus cycle (100 in this example) if needed. On the first figure below, we see a simple movement of the drone, and on the second figure we may see the movement that would move the drone out of the right border, and instead it is "wrapped around" to the left border of the game board.

assign3/ass3_fig2.pngassign3/ass3_fig3.png

How to calculate a field-of-view of a drone on the game board
Note that each drone has its current position defined by coordinates x, y, and an angle α: (x,y,α). We need to calculate a field-of-view of a drone in order to detect if a drone may destroy a target. The figure below shows the field-of-view of the drone. We define a filed-of-view as two lines of angle ±β from the drone’s current angle.
assign3/ass3_fig4.png

How to detect if a targes is in drone's field-of-view
Here you should use your scales of float point calculations in Assembly. Let the drone be at coordinates (x1, y1) with the current angle alpha. Let the target be at coordinates (x2, y2). Let maximum detection diatance be d, and let detection spread angle be beta.
Compute angle
gamma = arctan2(y2-y1, x2-x1)

The drone may detect and destroy the target iff:

(abs(alpha-gamma) < beta) and sqrt((y2-y1)^2+(x2-x1)^2) < d

Note that you may need to do modulus on the angle: if the difference in angles is greater than pi, add 2*pi to the smaller angle before doing the subtraction. This calculation would work as long as beta < pi/2, which you may assume.

How to generate a pseudo-random number
You should use Linear-feedback Shift Register method to get a random number in Assembly. A linear-feedback shift register (LFSR) is a shift register whose input bit is a linear function of its previous state. LFSR is a shift register whose input bit is driven by the XOR of some bits of the overall shift register value. The initial value of the LFSR is called the seed, and because the operation of the register is deterministic, the stream of values produced by the register is completely determined by its current (or previous) state. We would use Fibonacci LFSR version. The figure below describes a 16-bit Fibonacci LFSR. The feedback tap numbers shown correspond to a primitive polynomial in the table, so the register cycles through the maximum number of 65535 states excluding the all-zeroes state. The state shown, 0xACE1 will be followed by 0x5670.
assign3/ass3_fig8.png

The bit positions that affect the next state are called the taps. In the diagram the taps are [16,14,13,11]. The rightmost bit of the LFSR is called the output bit. The taps are XOR'd sequentially with the output bit and then fed back into the leftmost bit. The sequence of bits in the rightmost position is called the output stream. The bits in the LFSR state that influence the input are called taps.

Note that the XOR function is as follows:

assign3/ass3_fig9.png

Where we use Threads in the game
The drones and target are managed by threads, one thread each. We call thread a cooperating routine, or co-routine for short. In addition, we have a co-routine for the scheduler that manages when to run the other threads. A specialized co-routine called the printer prints the current state of the game board. A drone’s co-routine number is its id, which should be visible to it as an argument at activation (either in a register, or on its own stack, your choice). We use drone’s id for scheduling and for printing. The program begins by creating the initial state configuration of drones and the initial state of the target. The program initializes appropriate drones and target, and control is then passed to a scheduler co-routine which decides the appropriate scheduling for the co-routines. The scheduling algorithm for drones is ROUND ROBIN, meaning that co-routines are scheduled in a loop: 1,2,3,…,N,1,2,3,…N,… and so on. The printer co-routine should print the current state of the game board each K steps, where step means an execution step of one drone.
The function of a drone co-routine is as follows:

(*) calculate random angle ∆α       ; generate a random number in range [-60,60] degrees, with 16 bit resolution
(*) calculate random distance ∆d    ; generate random number in range [0,50], with 16 bit resolution        
(*) calculate a new drone position given ∆d and ∆α as follows:
    (*) first change the current angle to be α + ∆α, keeping the angle between [0, 360] by wraparound if needed
    (*) then move ∆d at the direction defined by the current angle, wrapping around the torus if needed
(*) call mayDestroy(…) to check if a drone may destroy the target
(*) if yes
    (*) destroy the target	
    (*) if number of destroyed targets for this drone are >= T	
       (*) print “Drone id <id>: I am a winner”	
       (*) stop the game (return to main() function or exit)
    (*) resume target co-routine
(*) if no	
    (*) switch back to a scheduler co-routine by calling resume(scheduler)	
The function of a target co-routine is as follows:

(*) call createTarget(...) function to create a new target with randon coordinates on the game board
(*) switch to a scheduler co-routine by calling resume(scheduler) function
The loop in the function of a scheduler co-routine is as follows:

(*) start from i=1
(*) switch to the i’s drone co-routine
(*) i++
(*) if i == K there is a time to print the game board
    (*) switch to the printer co-routine
The function of a printer co-routine is as follows:

(*) print the game board according to the format described below
(*) switch back to a scheduler co-routine
Command line arguments
Note that the game border size is pre-defined to be 100 x 100.
Your program should get the following command-line arguments (written in ASCII as usual):

N – number of drones
T - number of targest needed to destroy in order to win the game
K – how many drone steps between game board printings
β – angle of drone field-of-view
d – maximum distance that allows to destroy a target
seed - seed for initialization of LFSR shift register

> ass3 <N> <T> <K> <β> <d> <seed>
For example: > ass3 5 3 10 15 30 15019

Initial configuration of the game
Initial configuration is calculated (pseudo)-randomly with 16 bit resolution (for each required number) before the game begins, by the main() function. What this means is: generate a 16-bit pseudo-random integer by using the LFSR to generate 16 new pseudo-random bits (for each required number, shift the register 16 time, computing a new radnom bit per shift), and scale this pseudo-random number to fit the desired range. All coordinates should be inside the board (may be also on the board borders). Initial angle values are floating point in the range [0,360], but note that you need to convert angles into radians if you want to use the SIN, COS, or SINCOS instructions (recommended). All other values are also float point numbers.
You should allocate a dynamic array of drones, which would contain drones’ current positions, headings, and scores. Every co-routine will be able to access this array for reading and writing. After reading the arguments and the file, you need to allocate space and initialize the co-routine structures. Two additional co-routines should be initialized: scheduler and printer.

How to print the game board
Each K drone steps the game board should be printed by printer co-routine. All floating point numbers to be printed using %f or %lf, all angles to be printed in degrees (for readability). Each printed line should end with a newline character. The printing is in the following format:
x,y	                               ; this is the current target coordinates
1,x_1,y_1,α_1,numOfDestroyedTargets    ; the first field is the drone id
2,x_2,y_2,α_2,numOfDestroyedTargets    ; the fourth field is the number of targets destroyed by the drone
…
N,x_N,y_N,α_N,numOfDestroyedTargets
Important implementation requirements
A scheduler co-routine MUST be exclusively written in a separate file, the actual control transfer (context switch) should be done with the resume mechanism. Hence, label resume would be declared as extern to the scheduler, and register ebx would be used to transfer control.
Different implementations (not ROUND ROBIN) of the scheduler will be examined by the checkers
Make all possible variables global in order not to pass them as arguments to your functions
