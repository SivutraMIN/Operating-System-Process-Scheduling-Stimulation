# Operating-System-Process-Scheduling-Stimulation

Computer Science A Final Project Report

1. Idea
Our project is an OS Process Scheduling Simulator: a desktop application, built with Python and Tkinter, that lets a user create a batch of CPU processes (each with an arrival time, burst time, and priority), run that batch through five classic CPU scheduling algorithms, and immediately see the results as a color-coded Gantt chart together with a metrics table of completion, turnaround, and waiting times.
We chose this topic because CPU scheduling is one of the core Operating Systems concepts covered in this course, and it is usually taught through hand-drawn Gantt charts and manual arithmetic on the whiteboard. That approach makes it easy to make small calculation mistakes and hard to compare how different algorithms behave on the exact same workload. Turning the algorithms into working code forced us to pin down every edge case precisely (ties, idle CPU time, processes that have not yet arrived, preemption, etc.) in a way that reading about scheduling in a textbook does not, and building a GUI around it let us turn an abstract topic into something interactive that we and our classmates could actually experiment with.
The five algorithms implemented are: First-Come, First-Served (FCFS), Shortest Job First – non-preemptive (SJF), Shortest Remaining Time First – preemptive SJF (SRTF), Priority Scheduling – non-preemptive, and Round Robin (with a user-configurable time quantum).
2. How the Chosen Project Will Benefit You or Society
For us, building this project reinforced several concepts from this course at once: dataclasses and object-oriented data modeling, algorithm design (greedy selection, simulation loops, preemption), and building a real, usable GUI with Tkinter (widgets, layout managers, event binding, and canvas drawing). It also gave us practice separating a program into clean layers — a data model, a computation engine, and a presentation layer — which is a design habit that transfers directly to larger software projects.
More broadly, the tool has value as a study aid: any student taking an Operating Systems course can use it to generate practice problems (via the auto-generate feature), run them through each algorithm, and immediately check their own hand-worked Gantt chart and average waiting/turnaround times against the simulator's output. Instructors could use it the same way during lecture to demonstrate, in real time, how the same set of processes behaves completely differently depending on which scheduling policy the OS uses — which is normally hard to show live on a whiteboard. Because the scheduling engine is decoupled from the GUI, it could also be extended later (e.g. to a web app, or with additional algorithms such as Multilevel Queue or Earliest Deadline First) without rewriting the core logic.
3. How Did You Do It?
3.1  Requirements
The application needed to: (1) let a user build a list of processes manually or generate one randomly; (2) compute the schedule for a chosen algorithm, including a configurable time quantum for Round Robin; (3) display each process's completion, turnaround and waiting time, plus the batch averages; and (4) visualize the resulting CPU timeline as a Gantt chart, including idle CPU periods, in a way that stays readable even when the chart is wider than the window.
3.2  System Design
The program is organized into three layers, written in a single file (SimpleVersion.py) but kept logically separate:
•	Data Model — a Process @dataclass holding pid, arrival_time, burst_time, priority, and the fields that get filled in once the process has been scheduled: remaining_time, completion_time, turnaround_time, and waiting_time. __post_init__ initializes remaining_time = burst_time.
•	Scheduling Engine — a BatchScheduler class whose five algorithms are implemented as @staticmethod functions with no shared mutable state. Each method clones the input list first (_clone) so the engine never mutates the caller's original processes, runs the algorithm, and returns a tuple of (scheduled_processes, gantt_chart).
•	Presentation Layer — a BatchSchedulerGUI class built with Tkinter/ttk: input widgets for adding processes, a Treeview table for the process queue and metrics, and a Canvas for the Gantt chart.
This separation means the scheduling logic can be tested or reused independently of the GUI, and the GUI only has to know how to call BatchScheduler and render whatever it returns.
3.3  Data Structures & Algorithms
Each algorithm returns a Gantt chart as a list of (pid, start_time, end_time) tuples. For the tick-by-tick algorithms (SRTF and, internally, the queue loop in Round Robin) consecutive 1-unit slices belonging to the same process are coalesced by a shared helper, _merge_gantt, so the chart shows clean blocks instead of dozens of 1-unit rectangles.
•	FCFS — sorts processes by (arrival_time, pid) and runs them strictly in that order, inserting an IDLE block whenever the CPU would otherwise sit empty waiting for the next arrival. O(n log n).
•	SJF (non-preemptive) — at every decision point, scans all arrived-but-not-completed processes and picks the one with the smallest burst_time (ties broken by earliest arrival), runs it to completion, then repeats. O(n²) in this implementation.
•	SRTF (preemptive SJF) — advances the simulation one time unit at a time; at each tick it picks the arrived process with the smallest remaining_time (ties broken by arrival time) and executes it for exactly one unit, which is what allows a newly arrived shorter process to preempt the one currently running.
•	Priority (non-preemptive) — identical structure to SJF, but the selection key is priority instead of burst_time (lower number = higher priority, per the Process model).
•	Round Robin — maintains a FIFO queue of ready process indices. It pops the front of the queue, runs it for min(quantum, remaining_time), appends any processes that arrived during that slice to the queue, and — if the running process is not finished — re-enqueues it at the back. An idle period is inserted if the queue empties before the next arrival.
For every process, once it finishes, the engine computes: completion_time (when it finished), turnaround_time = completion_time − arrival_time, and waiting_time = turnaround_time − burst_time — the standard OS definitions.
3.4  User Interface
The GUI (Tkinter + ttk) is split into three panels:
•	Configuration panel — entry fields to add a single process (PID, arrival, burst, priority), an “Auto-Generate” control that creates a random batch of 2–50 processes, a dropdown to pick the algorithm, a quantum field that is only enabled when Round Robin is selected, and a “Run Simulation” button.
•	Process table — a ttk.Treeview listing every process with its arrival/burst/priority, and — after a run — its completion, turnaround, and waiting time, plus a label showing the batch averages.
•	Gantt chart — drawn on a tk.Canvas: each scheduled block is a colored, outlined rectangle sized proportionally to its duration (with a minimum width so short slices stay visible), labeled with the PID and duration when there is room. Idle time is shown in grey. A time axis below the blocks marks every boundary timestamp, staggering overlapping labels onto two rows so they don't collide, and the whole chart sits inside a horizontally scrollable frame so long schedules remain readable.
Each process id is assigned a random pastel color the first time it appears and reuses that color for the rest of the run, so the same process is visually consistent across the Gantt chart.
3.5  Testing & Results
Because the scheduling engine is a set of pure, stateless static methods that don't touch Tkinter, we could test it directly from a script, independent of the GUI. We ran a fixed, hand-checkable batch of four processes through every algorithm and verified the engine's output against the results worked out by hand using the standard OS scheduling formulas:
Process	Arrival Time	Burst Time	Priority (1 = highest)
P1	0	5	3
P2	1	3	1
P3	2	8	4
P4	3	6	2

Running this exact batch through the simulator produced:
Algorithm	Execution Order (Gantt Chart)	Avg. Turnaround Time	Avg. Waiting Time
FCFS	P1(0-5) → P2(5-8) → P3(8-16) → P4(16-22)	11.25	5.75
SJF (non-preemptive)	P1(0-5) → P2(5-8) → P4(8-14) → P3(14-22)	10.75	5.25
SRTF (preemptive)	P1(0-1) → P2(1-4) → P1(4-8) → P4(8-14) → P3(14-22)	10.50	5.00
Priority (non-preemptive)	P1(0-5) → P2(5-8) → P4(8-14) → P3(14-22)	10.75	5.25
Round Robin (q = 2)	P1→P2→P3→P1→P4→P2→P3→P1→P4→P3→P4→P3	15.25	9.75

These numbers match the values obtained by solving the same batch by hand, which confirms the engine implements the standard formulas correctly. As expected for this workload, SRTF produced the lowest average waiting time (preemption lets a late-arriving short job jump the queue), FCFS the highest among the non-Round-Robin algorithms (because the long P3 blocks everything behind it once it starts), and Round Robin — despite being the fairest algorithm in terms of response time — has the worst average turnaround/waiting time here because of the context-switch-like overhead of repeatedly re-queuing partially finished processes.
We also used the GUI's auto-generate feature to stress-test the engine and interface with batches of up to 50 randomly generated processes, confirming that: ties in arrival time, ties in burst time/priority, gaps between arrivals (idle CPU), and processes that arrive after the simulation clock has already passed zero are all handled without crashing, and that the Gantt chart's horizontal scrolling and label-staggering logic keeps the chart readable even with many short slices.
3.6  Challenges
•	Getting SRTF and Round Robin to produce a clean Gantt chart required simulating the schedule one time unit at a time and then merging consecutive same-process slices back together (_merge_gantt) — without that step the chart would show dozens of tiny 1-unit rectangles instead of readable blocks.
•	Breaking ties consistently (same burst time, same priority, or the ready queue emptying before the next arrival) took care to avoid starving a process or desynchronizing the Gantt chart from the metrics table.
•	Laying out the Gantt chart's time-axis labels so they don't overlap when many short slices are packed close together required detecting when two labels would collide and staggering them onto two rows.
3.7  Reflection
Building the scheduling engine before the GUI made the project much easier to reason about: once the five algorithms were correct and independently testable, wiring them into Tkinter was mostly a matter of formatting their output. Keeping BatchScheduler free of any GUI code also means it could be reused as-is in a different front end (a web app, a CLI, or a notebook) without any changes. If we extended this project further, we would add an automated test suite (e.g. pytest) around the engine instead of relying on manual spot checks, and consider adding more algorithms (Multilevel Queue, Multilevel Feedback Queue) and the ability to export a run's results/Gantt chart as an image or CSV.
3.8  Tools & Generative AI Usage
	AI is used to do the presentation layer of the program, in other words, the logic and how the program work is written by a human and the UI is created by the AI.
4. Appendix
Source Code
The complete, well-structured source code is included as a separate, zipped folder alongside this report, as required by the submission guidelines.
Generative AI Usage Disclosure
AI is used to do the presentation layer of the program, in other words, the logic and how the program work is written by a human and the UI is created by the AI.
<img width="451" height="702" alt="image" src="https://github.com/user-attachments/assets/0442a74b-4c82-4ad5-b83e-f29c97699e42" />
