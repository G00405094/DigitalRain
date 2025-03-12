---
layout: post
title: A Project in Modern C++
tags: cpp coding project
categories: demo
---

Project Overview
The project is split into three main files:

DigitalRain.h
Contains the class declaration. This file defines the DigitalRain class, which manages all aspects of the animation—from generating random characters to drawing them on the screen. It includes constructors (default, parameterized, and copy), a destructor, and even an overloaded operator+ to merge two DigitalRain objects, showcasing some basic operator overloading.

DigitalRain.cpp
Implements everything declared in the header file. This is where the magic happens. The code sets up a screen buffer (using a vector of strings), initializes the random number generator, and launches asynchronous tasks. Each column of the digital rain is animated by a separate thread, creating an independent "drip" effect. A dedicated drawing loop continuously refreshes the screen with the updated buffer, ensuring a smooth animation.

main.cpp
Serves as the entry point for the program. It creates an instance of the DigitalRain class, starts the animation, waits for the user to press Enter, and then gracefully stops the animation.

Key Features
Multi-threaded Animation:
Each column of falling characters is handled by its own asynchronous task. This means that the rain effect is not only visually appealing but also a practical example of how to use multithreading in C++.

Robust Randomness:
Instead of using the outdated rand() function, DigitalRain uses the Mersenne Twister (std::mt19937) to generate truly random characters. This ensures a better distribution of characters and a more natural rain effect.

Dynamic Screen Buffer:
The screen is represented as a vector of strings, with each string representing a row of characters. This simple yet effective design allows the program to easily update and redraw the screen.

Thread Safety:
Since multiple threads are updating the screen buffer concurrently, the project uses mutexes (std::mutex) to ensure that these updates are done safely without any conflicts or data corruption.

Customizable Animation:
You can tweak several parameters:

Screen Dimensions: Set the width and height of the animation.
Drip Speed & Tail Length: Adjust the speed of the falling characters and the length of the trailing effect.
Spacing: Characters can be printed with extra spacing to change the overall look of the digital rain.
Operator Overloading:
As a demonstration of object-oriented programming, the operator+ function merges two DigitalRain objects by combining their screen buffers in an alternating pattern.

How It Works
Initialization:
When you create a DigitalRain object, the constructor sets up the screen buffer with the specified dimensions and seeds the random number generator.

Starting the Animation:
The start() function sets an atomic flag to true and spawns a thread for each column using std::async. Each thread runs the columnWorker function, which continuously updates a column with random characters, creating the illusion of falling text. Simultaneously, a separate drawing thread runs the drawLoop() function to refresh the screen at regular intervals.

Animating the Rain:
In the columnWorker, the current position in the column is updated with a new random character, and a "tail" (a few trailing characters) is cleared to create a drip effect. Once a column reaches the bottom, it loops back to the top, ensuring the rain effect is continuous.

Drawing the Screen:
The drawLoop() repeatedly calls drawScreen(), which clears the terminal and prints the entire buffer with ANSI escape codes to provide a green color, reminiscent of the Matrix.

Stopping the Animation:
When the user presses Enter, the stop() function is called, setting the atomic flag to false. This causes all worker threads to finish their execution, and the program exits cleanly.

<img src="https://raw.githubusercontent.com/G00405094/DigitalRain/main/docs/assets/images/RainImage.jpg" width="400" height="300">
