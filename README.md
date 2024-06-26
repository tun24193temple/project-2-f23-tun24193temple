<p align="center">
  <a href="" rel="noopener">
 <img width=200px height=200px src="https://i.imgur.com/6wj0hh6.jpg" alt="Project logo"></a>
</p>

<h3 align="center">Fabrication Plant Simulation Using Thread Synchronization</h3>

---

<p align="center"> Few lines describing your project.
    <br> 
</p>

## 📝 Table of Contents
- [About](#about)
- [Getting Started](#getting_started)
- [Implementation](#implementation)
- [Testing](#testing)
- [Usage](#usage)

## 🧐 About <a name = "about"></a>
The purpose of this project is to learn and experience multi-process and multi-thread development. Additionally, it is an excellent introduction to the use of locks for mutual exclusion and critical section control as well as the use of condition variables for synchronization control in concurrent execution and the use of file I/O shared among processes and threads.

## 🏁 Getting Started <a name = "getting_started"></a>
To understand the problem we must break down all parts of the problem. These parts are the Fabrication Plant Manager, the Assembly Manager, and the I/O files.
### I/O files:
1. Input file: Represents the railway car that arrives at the plant with parts. The railway car is generated by a separate C process I created that randomly generates a number between 1 and 25 and writes it into a .txt file. These numbers are then followed by a '\n' so that there is a new line between each part number. For my implementation, I generated 500 entries.
2. Output files: These represent the delivery trucks exiting the fabrication plant after receiving assembled parts. Each entry in these truck files will have a sequence number and a part number.
### Fabrication Plant Manager
1. Manages the I/O files by opening the railway car file to 'read' and the Output truck files as 'write'.
3. Creates a child process called the "Assembly Manager" which is responsible for reading from the input file and creating entries to put into the output files.
4. When the Assembly Manager signals completion, the fabrication manager shuts down the plant by closing all the files and terminating the process.
### Assembly Manager
1. Setup Area: We will have two threads in this area that are responsible for reading and sorting parts taken from the railway.txt file onto a conveyor belt. We will implement parts as a struct with their own sequence number and part number associated with them. The conveyor belt will also be implemented as a struct resembling a FIFO queue. Both threads have the same functionality but one can read and sort more parts at a time and can do this process faster than the other thread. i.e, thread L reads and sorts 25 at a time taking 0.25 seconds and thread R does 15 at a time every 0.5 seconds.
2. Assembly Area: This is where parts are taken off the conveyor belt and are processed for delivery. i.e. written to the delivery truck output files as entries containing a sequence and part number. 
### Prerequisites
- GCC compiler
## Implementation <a name = "implementation"></a>
### Fabrication Plant Manager
Since this plant is responsible for the creating, opening, and closing of the I/O files this file is relatively small just containing a main function. This function will first open the railway_car.txt file as read only as well as create the red_truck.txt and blue_truck.txt files. It will also create char[] file descriptors representing each file stream. Afterwards, it will call fork() and exec() the child process. It will wait for the child process to finish and then it will close all the files and terminate the process.
### Assembly Manager
#### Structs
##### Part Struct: Used to Represent an assembled part as {seq num, part num}
Used to assign a sequence number to the part number. A raw part is just an int data type between 1 and 25. Once assembled, it is turned into an assembled Part entry containing it's original part number (1-25) and an associated sequence number which is first initialized as 0 but once a thread in the child process accesses the first raw part from the railway car file, the sequence number is incremented and assigned to a Part struct.
##### ConveyorBelt Struct: Used to represent a FIFO queue
1. 'Part *buffer' is a dynamic array of type Part to simulate the conveyor belt as a FIFO queue.
2. 'int size' represents the maximum amount of parts the belt can hold.
3. 'int count' to track the current number of parts in the queue.
4. 'int in' is used to indicate where the next part will be placed in the queue.
5. 'int out' is used to indicate where the next part will be removed from the queue.
6. 'pthread_mutex_t mutex' ensures only one thread accesses the queue at a time to prevent a race condition.
7. 'pthread_cond_t not_empty' is a conditional used to signal waiting threads when the queue is not empty (i.e, there's at least one part that can be retrieved).
8. 'pthread_cond_t not_full' is a conditional similar to the conditional above but it signals to threads that want to place parts into the queue.
#### Put and Get Functions
##### void put(ConveyorBelt *cb, Part part):
1. Takes a ConveyorBelt pointer and a Part as parameters
2. To prevent a race condition, the put function first locks the conveyor belt mutex to prevent threadR's access if it desires to run the put function at the same time as threadL.
3. Waits for the conveyor belt to free up space using a condition lock
4. Once there is room on the conveyor belt i.e. the count is less than the size of the conveyor belt, the part is inserted into the 'in' index of the conveyor belt buffer and the in index is incremented along with the counter.
5. Finally, put() signals that there are contents in the conveyor belt using the not_empty conditional signal and unlock the mutex lock so other threads can access the conveyor belt.
##### Part get(ConveyorBelt *cb): 
Unlike the put() function, the get() function is of type Part and takes only a Conveyor Belt as a parameter. get() has a similar structure to the put() function.
1. First, get() acquires the mutex lock for the conveyor belt.
2. Wait until there are parts on the conveyor belt
3. Sets a Part value to the part on the conveyor belt that is on the out index.
4. updates the out index and decrements the count of parts on the conveyor belt.
5. Returns the part
#### Threads
##### ThreadL/R: 
threadL and threadR both take part numbers received from the railway_car.txt file and associate a sequence value to them to 'assemble' them as 'parts' but at different rates. ThreadL reads 25 part numbers from the railway car every 0.25 seconds and threadR reads 15 every 0.5 seconds.
1. Uses a buffer to store the part number as a string read from the railway_car.txt file.
2. Implemented a while loop inside and an infinite while loop. The while loop runs until 25 or 15 parts have been processed.
3.  I keep track of the part using a counter, each time the counter increments is when a part is put on the conveyor belt.
4.  Since we read one character at a time, a conditional statement is added so that the loop starts at the next iteration if the next character is not a new line indicating that we are still reading a part number.
5.  After we convert the buffer that stores the character(s) read from the railway_car.txt file into an integer and we reset the buffer in index to 0;
6.  Then we create a Part using the current sequence number and part number. We then increment the sequence number and also make sure to lock and unlock the sequence mutex so that another thread is not accessing the sequence number.
7.  Then we use the put function to put the part onto conveyor belt blue if the part number is 12 or less or onto conveyor belt red if the part number is 13 or more.
##### ThreadX/Y:
Thread X and Y are the consumer threads that take the parts produced by ThreadL and ThreadR off of the conveyor belt and onto the truck output files. Their function is the same except threadX writes to the blue truck and threadY writes to the red truck
1. Consists of a buffer that holds the Parts obtained from the conveyor belt
2. Inside an infinite while loop, I use the get() function to obtain the part on index 0 of the conveyor belt.
3. Then, I use the snprintf() function to format the part as a string: {seq num, part num}. 
4. The snprintf() function also returns an int for the new length of the string which is then used in the write() function to write the assembled part to the output truck files.
## 🔧 Testing <a name = "testing"></a>
### Testing Option Parsing Function
#### Testing I/O Files Creation
1. Main function when creating the trucks
* ![Main](Images/test_create_trucks.png)
2. Output
* ![Output](Images/test_truck_files.png)
#### Testing Conveyor Belt Initialization:
* ![Init](Images/testing_cb_init_input.png)
* ![Output](Images/testing_cb_init_output.png)
#### Testing Put/Get Functions
* ![Functions](Images/testing_cb_put_and_get_functions.png)
* ![Output](Images/testing_cb_put_and_get_output.png)
#### Testing ThreadL/R
* ![Output](Images/test_threadLR_function.png)
* ![Output](Images/test_threadLR_inside_threads.png)
* ![Output](Images/test_threadLR_main.png)
* ![Output](Images/test_threadLR_output.png)
### Frequent Errors While Testing
1. The first problem was figuring out how to implement the part and conveyor belt logic. Starting this project, I just had a series of global variables for the queue logic of the conveyor belt and vars to indicate the sequence and part number of a part. Because of this, I had way too many global variables since I had to make separate variables for the blue and red queues. This made programming the get-and-put functions challenging and implementing them into the threads even more challenging. I decided to turn the Conveyor belt into a struct and a part into a struct as well to better implement the get-and-put functions which never seemed to work under the old implementation.
2. ThreadL and R was hard to implement as well as I could only read one character at a time for it to work properly. Reading two characters at a time meant that I would have to read the space after a number comprising of a single digit and it gave me unwanted results. I had to implement a buffer that would store the characters read one at a time.
3. During the testing of threadsL/R, the conveyor belt would often be populated by parts with a part number of 0. I was not sure what directly caused it but it had something to do with how I was reading from the input file. Specifically, it had something to do with my initial conditional. At first, all my logic was under an if(character =='\n') statement and I had an else if that inserted the character into the buffer. I opted to change the first conditional to if(character !='\n') and had a continued call under it so that it would continue to the next iteration and add the next digit to the part number buffer.
4. My last error was that the program would execute fine but never stopped running due to poor EOF markings for my threads. At first, I made sure to generate a railway_car.txt file with -1 at the end of the file so all I had to write was a conditional saying to break the infinite while loop if we reach a part number of -1. This worked fine for my L/R threads but since the get function needs a part that threads L and R weren't producing, threads X and Y would run indefinitely. Instead, I made threads L and R produce a part {0,0}, and when X/Y receives this part they break their while loops.
## 🎈 Usage <a name="usage"></a>
1. If you wish to make your own railway_car.txt file I provided a C file that generates that for you. Modify the for loop limit if you wish for more or less entries in the file.
2. Put all packages in the same directory along with the Makefile. Run 'make' in the Linux terminal and it should compile all the code.
3. Type './plant' to run the code.
