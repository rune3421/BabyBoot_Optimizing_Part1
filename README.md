# BabyBoot_Optimizing_Part1
Now that the device is working, it's time to optimize the datastream from all the sensors. 

Hello and welcome to another episode of Karl Fiddles Around™. 

In our last episode, we integrated the final part of the circuit, just to make sure a rough-cut version of the sensor set would work, and it did! Wiring and basic clunky software are done, so now it’s time to go back and look at optimizing. 

The main goal today is to clean up the datastream, so that the ECG gets the priority high speed flow it needs, without having to cut any of the other sensors out of the system. 

There’s a few things I’m going to try to do today, to find out which does the most good. 

A new friend of mine highly recommended getting every combination of sensors in the set through the wifi, to see where there might be a conflict or competition for resources. This seems to be very time intensive, but also pretty thorough. 

There are also a couple of preprocessing calculations in the code that I haven’t turned off yet, that are supposed to run every time the ECG is sampled, which could be moved to post-processing, if the data is clean enough. 

There’s also the fact that the Websocket is being called every time the ECG is sampled, which adds to its processing time. 

There’s also the fact that the client is attaching timestamps to the samples after they’ve gone through the websocket, which means there’s some batching going on, and that makes the time less accurate. Some of the other functions in the server side include a time measurement, so I can use those function calls for the timestamp on the ECG sampling, so it can get batched after that. 

There’s also the sampling rate of the other sensors that can be slowed down a lot; the server is sampling the temperature, pH and CO2 once a second for now, but at least the pH and temperature can be slowed down quite a bit. CO2 sampling rate is tenuous, because its accuracy might depend on a high sampling rate, but that’s also a topic for a later project. 

So, here’s the order of business for this round; 

Turn off the Heart Rate and Respiratory rate calculations, and check performance
Move the ECG timestamping from the client to server side, and check data
Slow down the sampling rate of the Temp, CO2 and pH sensors and check performance
Refactor the Temp, CO2 and pH function call to a separate function call with a timer in main loop, to separate the websocket function call from the sensor function call and check performance
Batch the ECG samples, and slow down the websocket cycle rate and check data
If all this fails, check each sensor combination two at a time until we find out which sensors are conflicting. 
With step one, we are technically losing one piece of information, so we’ll comment out the relevant sections of code…after saving this as a new version. 

And the compile worked, which means I edited out correctly, but there wasn’t a noticeable change in performance. So…I might undo that later. We’ll see. I’m glad I only commented it out, instead of deleting the info.

Next, it’s time to move the ECG timestamping. For this, though, I want to double check that I’ve got the most efficient timestamper available. I’ll test the millis() call, since I’m only really interested in the time since program start, as long as it’s accurate. 

Uploading ran into a few glitches, but solving that and running, and I still have the same performance issue, but I might as well take a data sample and take a look to see how the chart looks. 

Here’s with the time as x axis:

And the timer looks like it’s actually working now. We’re getting a very tight stream of timed data, so we can tell from the static data if there’s been an interruption. The waveform in this streaming section looks like a backfeed noise, which is interesting, but it means the signal is still coming through the ECG, but the interruptions are too long to get a real reading. We’ll keep this timer because it doesn’t add noticeable workload to the server, and it’s turning out to be useful for data cleaning and analysis. 

Next up it’s time to slow down the sample rate for the intermittent sensors. I’ll boost the interval to 50 times the original, and then take a look at the impact on the data. I’ve also noticed a function call in the JSON serialization step that might have interrupted the stream. I’ve moved that into the intermittent reading section, and changed the timing of the intermittent readings. 

And the interruptions don’t look much different. 

So there are still significant interruptions, and the ECG can run very fast between interruptions. A quick read through the charts confirms that the ECG can run at 200Hz, which is good. 

Resetting the intermittent sensors to 1Hz, it’s time to look at the next option. 

Refactor the Temp, CO2 and pH function call to a separate function call with a timer in main loop, to separate the websocket function call from the sensor function call and check performance. That compiled ok, but that was just an intermediate step to doing the next bit, which was batch the ECG samples, and slow down the websocket cycle rate and check data. 

In the process, I’ve found one more calculation that I might be able to turn off, so after I check this data, I’ll be able to:

Turn off FIR calculation
Then if that fails, go back through and check each sensor pair for conflicts and interrupts

That said, because I’m making a new buffer string that’s going to get passed all the way through the websocket and only unpacked once it gets to the .csv file, it’ll take a minute to debug the whole thing if I mess up the string or the passing procedures. 

So far so good on the compiling, now it’s time to see if the data comes out cleaner. If so, then I’ve narrowed down the scope of the problem and it’ll just be a matter of fiddling with the sequence and settings to optimize between string length and interrupt rate. 

And the data comes out null for the string argument I built to pass into the Json document, which means it’s time to study JSON data structures a bit and find a way to append to the document with each reading. 

To do that, I’m going to start a new blank arduino sketch and just include the sections of code I need to check, and merge it later once I get the sequence right. And, to sequence it, I need to have a switch/case setup. 

I think I need to put the sequence in the void loop, and just break out the larger chunks into callbacks. 

And, after taking a step back to have a vacation with the family, I’ve had some time to think about it in my subconscious, and I’ve realized that I might be over-focused on one possible problem, without diagnosing the problem first. In other words, the guy I met at the park was right. I need to go through the components one by one to find the problem first before designing solutions. I’m doing the ECG batching on the assumption it’s one of the sensors on the server side that’s interrupting the flow of data, and that batching will widen the stream in between sections of flow…which might not be the case. Looking at the data, it’s not interrupted at one second intervals, so it can’t be through the 1000millis function calls that the stream is interrupted. Instead, I’m going to need to look at the datastream at each stage. 

So, to start the diagnostic before the solving, I’m going to move the Serial.print through the different parts of the process to see if I can find where the interrupt happens. FIrst, the server side the moment the data gets sampled. 

To make sure the system is operating as close to the same state where the interrupt occurs, I’ve hooked up the client side to a power supply, so that the websocket is active and the SD card is writing while I check each part of the stream. 

Step 1 is to print to serial before transmission, to see if the waveform gets chopped downstream, and the signal sampling is actually continuous. 

And would you look at that! A continuous stream of sampling, straight to Serial like we wanted. So…something downstream HAS to be interrupting the flow.  It’s not the sensors. So the guy was half right. I have to check the components. At least I know from this one check that it’s not the upstream components (the sensors), but the downstream (either the websocket or the SD card) that’s interrupting the flow. So…

Step 2 is to print to Serial from the server side after JSON packaging. This one might not graph neatly, so I’ll just have to describe the printout to you as I see it. 

But whaddyaknow! Not only does it graph after JSON packing, it’s still a continuous stream of data!

So, it’s nothing on the server side (short of the websocket transmission) that interrupts the data flow. I’m starting to really doubt the SD card. Time to turn off the SD and print to serial from the client side. 

And…it is choppier. I don’t know for sure, but it looks like this could be the entire problem here. The Websocket connection isn’t 100% stable. The blue line at the top is the timer. You can see jumps in the timer, which indicate gaps in time where the serial plotter had nothing to print, but the time was increasing. The other interesting artifact is that the ECG signal flats out right before a time gap…which might indicate a slice of time where power is lost to the system and the ECG isn’t giving a true signal, right before cutting out entirely. The reason I think so is because, other than receiving this serial from the client side, the only other change is that the server is now plugged into a 9v and not getting power from Serial. 

So…a theory is emerging…but I have to turn on the SD card to find out if there’s a difference in the delay with it running. I’ll print to Serial one more time, but now it’s to see if the SD card causes more frequent or larger delays in the system. 

And…yes. It does look like there’s two separate delay sizes. One longer one that happens after a brownout on the server side, and a shorter, more regularly spaced one that happens without a brownout. So…the websocket is not the problem, but the SD card printing is. But so is the power supply on the server side, so yay!

You know what this means??? I can only solve one of these problems with software changes. I need to get a beefier power supply for the server side, most likely with a higher input amperage or just a separate Serial cable on a different computer (to not confuse COM ports for debugging), but I also have to get the SD card to write continuously without interruption. 

So, I have two tasks now; 

Change the SD writing loop so it streams continuously. 
Beef up the server power so there are no brownouts

For the SD card loop, I’ll add in two more cases to the websocket; one to open the file and one to close, so that the file is open for as long as the websocket is active. 

And it looks like we have our longer streams back, with only the brownout problem now!
But, because I modified the SD filewriting protocol, I’m going to have to make sure that the data is still getting through cleanly, which means comparing this chart to the SD chart. 

As I feared…no data got into the SD card. So…I’ll have to explore that issue another day. It’s time to undo that step, and see if we can get the brownout chunking problem solved by plugging the server into the COM port and have the client logging on a battery. 

Here’s the server running on COM power printing to Serial:

Nice and continuous stream. 

And here’s the client side logging on a battery:

Nevermind. Ran into an identical problem on the client side; the ESP would brown out every time the SD card tried to write…so the document would always come out blank! I’m probably going to need to add capacitors to both circuits, or do something to smooth out the power supply. 

But…that is a problem for a different Fiddle exercise. For now, we’ve solved the problem!
All the sensors and components are capable of streaming, except maybe the SD card. The main problem is power supply, for both the server and the client. Both draw too much power at certain points to be supportable by a single 9v. So…plugging both the server and client into USB power yields this:

From the client side serial. Notice there was a significant time cut without a brownout, and then a smaller time cut with a lag. 

This:

Is what got saved to the SD card. 

There are still significant interruptions in the stream, but there are no more brownouts on either side (according to the LED’s) which means although the power supply issue has been solved, there’s still a major issue with the data flow. 

Which led me to check back through my earlier versions, and watch the circuit much more closely. So, I’ve gone through a couple test cycles, and found some interesting things. 
Whenever the SD card is left inactive, the server side flows continuously without problem. This happens if the server is turned on AFTER the client. 
The Client doesn’t print anything to Serial if the SD card is inactive. 
If the server is turned on BEFORE, the SD card is able to write, but only intermittently. 
The server only samples sensors while the SD card is writing. 

This might mean there’s a backfeed from the client to the server through the websocket that is blocking the loop from progressing, originating in the SD card. So…let’s add in a couple debug messages to see where the error happens. 

It doesn’t look like the SD card is registering a failure opening the file…so let’s check the websocket. 

First thing to do is turn off Client handling. If there’s backfeed causing the server to pause, let’s cut off the backfeed and see if that fixes it. Commenting out server.handleClient(); didn’t fix it, so we’ll leave that in. 

Next we need to add some websocket type responses on both sides to see if we can catch what’s happening in the websocket layer that’s interrupting the server. 

All the possible WS types have been added to the server, and nothing pops up on the serial during interrupts. Now it’s time to do the same to the client side. 

And the client side doesn’t pop any errors during an interrupt either, which means someone of the three is lying. Does the SD have a hardware problem with a streaming write? Time for some research! Which means it’s also a good time to break and save this version of the system. Here’s the code for each side:

Server Code:
Client Code:

Aand, just because, I did a little extra where I tried taking the file open and close functions outside the loop and run the whole thing on a timer, closing the file after a set interval separate from the websocket loop. 

Here’s the chart:

Interruptions are still prevalent in the data, which means the open and close functions weren’t the problem. There’s still something else the SD card is doing wrong…somehow. More research needed! See you next time. 
