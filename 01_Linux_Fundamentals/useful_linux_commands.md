# Useful Linux Commands.

#### 1. Date : to display the current date and time we use the date command: `date`

![alt text](images/date.PNG)

---

#### To change the system date, you must login as a root user and change the date in the following format.

`date 12417052020.30`

- Explanation of above format is as follows:

![alt text](images/date_assignment.PNG)

---

### The Word Count Command in linux

#### 1. `wc file1`

- This will display the number of lines, words and characters in file.

  ![alt text](images/wc_command.PNG)

- if you want any specific thing then you just mention the mode and it will display only related information.

![alt text](images/wc_command_options.PNG)

---

#### 2. `cal` command: (current month calendar)

![alt text](images/cal_command.PNG)

#### 3. `cal -3` last, current and next calendar.

![alt text](images/calendar_command_option.PNG)

---

### 4. pwd (present working directory)

- It will show present working directory, that is full
  path of current location.
  ![alt text](images/pwd.PNG)

---

#### To see the huge data on the screen we use less or more command.

#### 1. The `more` command : The more is reading utility in Linux where you can see the data line by line or next screen but not in the up direction.

![alt text](images/more_command.gif)

---

#### 2. The `less` is the powerful command than `more` command because it goes in the both direction upward and downward to read the data.

- up arrow ⬆ to move upward direction to see the data.
- down arrow ⬇ to move downard directio to see the data.
- space bar key 󠀠󠀠➖ to move towards next screen.
- enter key ↵ to move towards next line.
- q key for quit

![alt text](images/less_command.gif)

#### One of the most controlled command to read the data and hence mostly used to read the big file data in Linux.

---

#### Variables in Linux are Key-Value pair

#### 1. variable example

- `flower=rose` : We are declaring variable flower having value as a rose.

- To check the variable value we use $ symbol infront of the variable to see the actual value of it.
  `echo $flower`

![alt text](images/variable_declaration.PNG)

---

#### The Linux also has the in-built variables as follows

- 1. $HOME : user's HOME value
- 2. $PWD : the current relative path

![alt text](images/in_built_linux_variables.PNG)

#### the `echo ~` command will show you the user's home directory, $HOME is abbereivated as tilde.

![alt text](images/tilde_home.PNG)

### Note:

#### The variables declared by the user are always in the "SMALL CASES" like `$flower`

#### The variables decalred by the system are always in the "UPPER CASES" like `$HOME`
