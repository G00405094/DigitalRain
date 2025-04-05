
## Introduction

In this project, I created the "digital rain" effect as a console animation in C++. Digital rain refers to the falling green characters made popular by The Matrix film. The goal was to recreate this effect in a Windows console, using modern C++ features like multi-threading, random number generation, and RAII based memory management. The result is an animation of characters falling in columns, with smooth visuals and stable performance. This blog will outline the design, algorithms, and lessons that I learned from building the Digital Rain simulator. Key components include a DigitalRain class that holds the animation logic (following Single Responsibility and encapsulation ​[4]), custom Pixel and Column structs for  the rain's characters and stream, and the use of double buffering to make the animation smooth. The project heavily involves using the C++ standard library (threads, mutexes, vectors, random engines, chrono timers), following best practices to use standard features instead of reinventing them ​[5]. Throughout development, I also wrote a runAllTests() function to check that object construction, copy/assignment operations, and exception handling all work correctly.

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/Screenshot 2025-04-05 075049.png" width="400" height="300">

## Design and Testing
The overall code design follows core C++ guidelines. Knowing that this code would be seen by other people, I included as many comments as I could, clearly describing whats going on so that the code is easy to follow for a fresh pair of eyes. 
Class Design: The core of the project is the DigitalRain class, which manages the rain animation. To follow good software engineering practices, it encapsulates all data and behavior related to the effect, only exposing the interface (startRain() to begin the animation) while hiding internal details​ [4]. This follows the Single Responsibility Principle: the class deals only with the simulation of the rain and nothing else. DigitalRain has two buffers (2D arrays of Pixel) to represent the screen. Each buffer is a std::vector<std::vector<Pixel>>, which is a dynamic 2D array for console rows and columns. Using std::vector for the buffers automatically manages memory, which uses RAII to avoid manual new/delete and prevent memory leaks​ [6]. 
I defined simple structs for the data elements:

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/pixel struct.png" width="400" height="300">

Pixel – represents a single character cell in the console. It holds the character to display and attributes like its color or brightness.

Column – represents a stream of characters (a falling column in the digital rain). It contains state needed for the rain logic, such as the column's x-position, the current head which is the newest character position in that column, and any timing or delay parameters for when the column resets or a new head appears.

By separating Pixel and Column, it makes the design modular and clear. Each structure has one purpose: Pixel for display data and Column for controlling the rain's behavior for that column. These are combined by the DigitalRain class, which tracks all rain streams, by storing a collection of Column objects.  

Double Buffering: To achieve smooth animation without flickering, I used double buffering. The DigitalRain class allocates two 2D buffers of Pixel, one acts as the current frame being displayed (front buffer) and the other is the next frame being prepared (back buffer). One buffer is shown on the console while the next frame is computed in the background. Once the next frame is ready, the buffers are swapped in a single step under a mutex lock for thread safety. Double buffering is a common technique in graphics to prevent tearing and flicker​[7]. The animation looks smooth to the viewer because one frame is drawn off-screen in a back buffer and is only switched to the front when it is ready. 

Multi-Threading Structure: Using threads to update the simulation and generate output was a key design choice. The DigitalRain class does not start these threads in its constructor. Instead, it provides a member function startRain() that spawns the worker threads for the update loop and render loop. This design keeps object initialization separate from starting complex behavior, adhering to a principle of not doing heavy work in constructors (which also makes it easier to test the object in isolation). Once startRain() is called, two std::thread objects are created: one runs the internal updateLoop() function and another runs the renderLoop() function. Each thread is given a specific role:

The update thread generates new frames, it updates the back buffer's Pixel data based on the rain algorithm (moving columns downward, spawning new characters, etc.).
The render thread outputs the current frame to the console: it takes the front buffer and writes each Pixel to the console at the correct position with the correct color.

Using std::thread makes it straightforward to run functions concurrently[1]. The class std::thread represents a single thread of execution in the program, so having two thread objects means two functions are running in parallel (one updating the simulation, one rendering)​[1]. This concurrency allows the program to prepare the next frame even as the current frame is being drawn. To prevent data races between the update and render threads, a std::mutex is used to guard the swapping of buffers and any shared data. A std::lock_guard<std::mutex> locks the mutex ensuring only one thread accesses the shared buffers at a time​[1]​. The lock guard automatically releases the mutex when it goes out of scope​, so I don't have to manually unlock it. 


Testing (runAllTests()): Alongside the main program, I added a runAllTests() function to verify that all components of DigitalRain work correctly. These tests cover object construction, copying, assignment, and exception handling:
Constructor and Destructor: Makes sure that a DigitalRain object initialises its buffers to the correct size and that no resources leak when destructed. 

Copy Constructor: The DigitalRain class defines a custom copy constructor to allow making a deep copy of the animation state. This was tested by copying an object and verifying that the new object has its own independent buffers. I also ensure that copying a not yet started DigitalRain is safe and that active threads are not unintentionally copied (std::thread is not copyable [1], so the class either avoids copying thread members or stops threads before copying).

Assignment Operator: Similar to copy constructor, I overload the assignment operator to properly assign one DigitalRain to another. It makes a copy of the source and then swaps internal data. This way, if any exception occurs during copying , the original object remains unchanged. Tests inject exceptions to ensure that a half copied object doesn't occur, either the assignment succeeds completely or the original stays intact, this follows C++ exception safety guidelines.

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/Tests.png" width="400" height="300">

Exception Handling: I simulate error cases such as attempting to start the animation on an excessively large console size or forcing memory allocation failures. The code is written to throw exceptions if it can't use necessary resources or if used incorrectly. The tests catch these exceptions to confirm that the error handling path works and that no resources (like console handles or threads) are left dangling. Because the design relies on RAII, if an exception is thrown during initialisation or copying, all owned resources are released automatically by destructors during stack unwinding​ [8]. This greatly simplifies error handling, e.g if buffer allocation failed, the std::vector would free any partially allocated memory on its own.

## Algorithm – Simulating the Rain

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/RenderLoop.png" width="400" height="300">

The animation logic simulates falling streams of characters across the console window. The console screen is a grid of characters. Each column on the screen behaves like an independent "rain stream" of characters that continuously falls from top to bottom. 
Here is how the algorithm works step by step:
Initialization: When the DigitalRain object is created, it initialises its internal state:
Determine the console width (M columns) and height (N rows), via the Windows Console API. Allocate the two buffers (currentBuffer and nextBuffer) as N x M grids of Pixel. Each Pixel initially might be a blank space or null character.
Initialise a list or array of Column structures, one for each column index 0 to M-1. For each Column, set starting parameters as a random start row for the first drop, a random delay before it becomes active, or any speed modifiers. For example, one column might start immediately with a character at row 0, while another might wait a few frames before starting a drop, to stagger the rain effect.
Seed the random number generator using std::mt19937 with a non-deterministic seed for unpredictable sequences.
Update Loop (Producer): The update thread continuously runs an updateLoop() function which updates the simulation. I use std::chrono::steady_clock to regulate this loop. A steady clock is a monotonic timer that won't jump or drift​[1]. 
In each iteration of the loop:
For each Column in the list:
Determine if the column currently has an active stream of characters falling. If not, decrement its wait timer or randomly decide if a new stream should start now (this makes those random gaps where a column is sometimes empty).
If the column is active:
Move the stream one step down: for that column , go through each row in the back buffer and copy the character from the row above it. Now what was at row i-1 in the previous frame becomes row i in the next frame for this column. This moves all characters down by one. The very top row of this column will get a new character generated at position [0, column_index] for the new head.
Use the random engine std::mt19937 together with a distribution to pick a random character for the new head. Mark the old head as part of the trailing stream.
If the stream's head has moved past the bottom, then the column's active stream ends.
After updating all columns' state into the nextBuffer (back buffer), the update loop gets the mutex lock (via std::lock_guard) and swaps the back and front buffer. 
The update loop releases the lock automatically when lock_guard goes out of scope and then uses std::this_thread::sleep_until() to wait until the next frame time. 

Render Loop (Consumer): The render thread runs a renderLoop() function in parallel with the update. Its job is to draw the contents of currentBuffer to the console screen each frame:
Each iteration, the render loop locks the mutex briefly to make sure it is reading a stable frame. When its locked, it reads the currentBuffer and outputs it to the console.
To output at specific positions, the code uses Windows console API functions. GotoXY(x,y) is used to move the console cursor to column x and row y before printing a character.The render loop iterates over all Pixel entries in the current buffer and prints them in the console window.
After drawing the full frame, the render thread also sleeps until the next frame time.
By running update and render in parallel, the program takes advantage of multi-core processors: one core can handle the physics/logic of the rain while another core prints the frame to screen. The leading character of each column (the head) is rendered in bright green (color code 10), while the trailing characters use a normal green (color code 2). This  gives the rain a glowing head that reflects that Matrix look. For smooth animation, I use two different timers: the update loop runs at around 20 frames per second (50ms intervals), while the render loop runs at 60 frames per second (16ms intervals).

## Problem Solving and Challenges
Developing this project presented several challenges that required problem-solving and application of C++ techniques:

Flicker Output: Writing characters directly to the console as they updated caused a bad flickering and tearing effect. I solved this by using double buffering. With one buffer dedicated to preparing the next frame and the other for display, the user dosen't see the flickering. The technique of front and back buffers made the animation smooth[9].

Concurrency and Data Races: The frame buffers share data which brought the risk of race conditions[9]. Without proper locking, there's broken looking characters and inconsistent state which are signs that the update and render threads were getting in the way of each other. The solution was to introduce a std::mutex, when swapping buffers. By using std::lock_guard, I applied a lock for the duration of those operations​. This guaranteed mutual exclusion​[1] and got rid of race conditions[9]. 

Precise Timing: I wanted the rain to fall at a consistent rate, independent of machine speed. Using std::this_thread::sleep_for in a loop can cause error over time. To solve this, I used std::chrono::steady_clock and sleep_until(). The steady clock is monotonic and really good for measuring intervals​[1][9]. By computing the exact target time for the next frame and sleeping until that time, there's a stable frame rate. 



Windows Console API: Using the Windows-specific console functions for cursor movement and coloring was a new experience. Functions like SetConsoleCursorPosition and SetConsoleTextAttribute come from the <windows.h> API. I encountered some challenges such as the console flickering when clearing the screen. By directly positioning the cursor for each character, I avoided using full screen clear commands, which helped reduce flicker and prevented the screen from scrolling. 

Following C++ Best Practices: Throughout the project, I made an effort to apply the principles from my lectures and C++ guidelines. I made sure to make classes with encapsulation (all data in DigitalRain is private, changed only through its member functions). Another example is following "use the standard library before reinventing features"​ [5]. I used std::thread and std::mutex instead of any platform specific thread code[9], and std::mt19937 engine and distributions instead of custom random logic. Using the standard library also makes it easier for anyone reading the code, since things like vector and thread are familiar from cppreference.

Another small detail was handling the console cursor. I hide the cursor during rendering using the Windows Console API's CONSOLE_CURSOR_INFO structure and SetConsoleCursorInfo function. Without this a cursor would appear blinking during the animation. The cursor is restored to visible when the animation stops. 

To minimize mutex lock, both the update and render threads create local copies of the buffer data. The update thread prepares an entirely new frame in a temporary buffer before getting the mutex only briefly to swap buffers. The render thread makes a local copy of the current buffer under a short lock, then releases the mutex before the process of drawing to the console. This ensures that locks are held for the absolute minimum time possible, allowing both threads to run independently for most of their execution[9].

Each challenge above taught me something valuable. The debugging of race conditions showed me how important thread synchronization is. Implementing double buffering gave me insight into some of graphics programming, I learned why games use this technique and saw its effect first hand. Exception safety and correct copying was a good exercise in resource management and the Rule of Five, which really improved my understanding of C++ object semantics (copy, move, destroy).

## Reflection
Building this project was a rewarding experience that really improved my C++ coding skills and project development skills overall. The project covers a wide range of topics, mianly including ones that I had recently learned in class. It was really nice to be able to apply these topics in a practical way and think about how to implement them. The project was a great way for me to understand how something like vectors and threads can be applied. It was fullfilling to be able to try different approaches of the application out first to see what looked best and what I liked working with. Alot more time was spent researching things (like the double buffer) than I had initially anticipated. Researching new techniques and C++ functionality i'm unfamiliar with is really good practice and is a transfereable skill that I can take with me into the working world. So I was happy to take my time with these new topics I was learning and really try to get a grasp with it. reading documentation as well as going to tools like ChatGPT really helped to apply what I knew to the best of my ability. ChatGPT gave me the initial idea for the double buffering technique, but I didn't attempt to work with it until I had a classmate who was working with something similar, explain it to me. Using these AI tools to help integrate these techniques into my specific project is something that is new to the programming world but seems to be something that programmers are more and more expected to be able to work with. Working with AI tools was an interesting as it could help me in some parts(double buffering suggestion), but when it came to applying the techniques and coding style I was learning in class, it wasn't as helpful as it didn't have the context of what I was working on in these classes. It came in best as a tool to help when I was at a roadblock in the project and would't know what to do next or how to fix a certain problem. I found so much value in debugging different steps of the project as it put the principles of what I was learning in class about clean maintainable code into context. Those situations is where the point of projects like these come to light, which is appying what I've newly learned and discovering new technology and skills that I can apply in the future. Overall I really enjoyed this project and was grateful for everything I learned from it. I intend to continue building projects like this into the future as they are incredibly useful for learning. 

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/Screenshot 2025-04-05 075049.png" width="400" height="300">

## Refrences
[1] CPP Reference
[2] Programiz  https://www.programiz.com/cpp-programming
[3] https://wiki.osdev.org/Double_Buffering#:~:text=,later%20and%20might%20be%20invisible
[4] Software and Engineering Principles Lecture
[5] MatrixClass slides
[6] Memory Slides
[7] https://brilliantsugar.github.io/posts/how-i-learned-to-stop-worrying-and-love-juggling-c++-atomics/
[8] Exceptions Slides
[9]ChatGPT
