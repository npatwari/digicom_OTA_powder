## Wireless Communication Lab Assignment: Over-the-air Narrowband QPSK Modulation and Demodulation

This tutorial goes step-by-step through the process of transmitting and receiving a narrowband digitally modulated packet over the air in [POWDER](https://powderwireless.net/). Also, it describes a student lab assignment that has them implement a QPSK receiver that synchronizes to, and demodulates, a QPSK-encoded packet.

Let's start with a big picture view of the software we use to run this tutorial.  
1. We create a modulated packet signal in Python, and save it as a file. 
2. We start an experiment on POWDER with a few software-defined radio (SDR) and compute nodes, and specify a quiet and available frequency band to operate in. 
3. We use the Shout measurement framework to automate and synchronize TX/RX operations across multiple POWDER nodes, specifying the particular parameters for our experiment in a JSON file.  Shout allows us to transmit from one node, and receive at all of the other nodes; and then to iteratively switch the transmitting node (and receiving nodes). Each reception results in a complex-baseband received signal, which are all recorded to file.
4. We use Python to read in the file (now offline) and operate the receiver.

### Create Transmitted Packet IQ File

Open a Jupyter notebook for `QPSK_siggen_digicom_OTA.ipynb`. Type a string in `message` that is your "secret message" that will comprise the data in the packet for your students to decode. In my code I use 7 bits per character, so 1) not all AASCI characters are possible, and 2) the code requires the message to be an even number of characters to work with QPSK since it requires two bits per symbol. Give the output file a unique name, which we'll refer to later as <TX-filename>. Then run it, it will save <TX-filename> in the local directory.
 
### Log in to Account

This tutorial assumes you have an account on POWDER and that you're logged in. For more information on obtaining an account, please see [POWDER](https://powderwireless.net/).

### Reserve Resources

Typically, the POWDER over-the-air resources are in high demand. If you want to make sure you're going to be able to run an experiment on any particular day, it's a good idea to reserve the resources you need. To do this, go to [Reserve Resources](https://www.powderwireless.net/resgroup.php) and reserve 2-4 nodes. 

Here's what I reserved for the below experiment.
- Time: 12pm - 4pm Monday, May 13.
- Emulab Nodes: 
  1. cbrssdr1-browning
  2. cbrssdr1-meb
  5. cbrssdr1-honors
- Frequency range: 3383-3393 MHz

My choice of nodes was based on the [current availability](https://www.powderwireless.net/resinfo.php). In addition what POWDER says is an available frequency, one should see if there is usually interference in that band. We don't want to compete with other strong signals in the band. Beyond trying to be nice to other wireless systems in the area, it is just harder to demodulate our signal if it is on top of some other wireless signal. To check, go to the [Powder Radio Info](https://www.powderwireless.net/radioinfo.php) page, and look in the "RF Scans" column. There is an icon link if this is a radio that is regularly measured. For example, look at the [Honors rooftop node scan](https://www.powderwireless.net/frequency-graph.php?baseline=1&endpoint=Emulab&node_id=cbrssdr1-honors&iface=rf0). In this plot you need to ignore the regular spikes (these are due to LO feedthrough). The title, which for me is "Spectrum Monitoring Graph for Emulab cbrssdr1-honors:rf0 02/06/2024 8:04:29 AM", says the cluster (Emulab), the node name (cbrssdr1-honors), the name of the antenna port (rf0), and the date and time when the scan was taken.  Look at least at the nodes you intend to reserve, and find at least a few MHz that does not look occupied at any of your nodes.

The spectrum availability will vary. We only need about 0.5 MHz for the experiment I run, but I want to 
- don't request a band another POWDER user is using,
- make sure my sidelobes don't exceed my reservation limits, and
- make sure I have some space in case there is some narrowband interference at the time I run my experiment to move within my reserved band.

### Instantiate an Experiment

When it is time to run your experiment, use "Experiments: Start Experiment" on the Powder website to begin.

2. *Select a Profile*. Use the "Change Profile" button and the search field to search for "shout-long-2024". the [shout-long-measurement](https://www.powderwireless.net/show-profile.php?profile=2a6f2d5e-7319-11ec-b318-e4434b2381fc) profile.  Click "Next" to advance to the next step.
 - If you do not have access to the profile, one can be created via the "Experiments: Create Experiment Profile". In this case, fork and modify the [existing profile repository](https://gitlab.flux.utah.edu/npatwari/proj-radio-meas). Use your new git repo as the "Source". Pick a name for your profile, and the project group (the group you are sharing it with), and click "Create".
3. *Parameterize*. Specifying all necessary parameters.
 - Leave the Orchestrator and Compute Node Types to the default.
 - On the “Dataset to connect” select “None”.
 - Under "CBAND X310 radios." I create five Radio sites in this category as I listed in the Reserve Resources section above. (Most are self-explanatory, but "bes" = "Behavioral" and "fm" = "Friendship Manor").
 - Under "Frequency ranges for CBAND operation" I put in my min and max frequencies, the number only, in MHz.
Click "Next" when finished.
4. *Finalize*. The name must be 16 charaters or less, so I put in my initials and some minimal abbreviations for what I'm doing. You want it to be unique so you can share it with someone else if they want to look at / debug it.  Select your project and Emulab cluster. Click "Next".
5. *Schedule*. Put in the end time (at the least) so that someone else can use the resources after you plan to be done. Click "Finish".

The setup wipes the machines and does a full install of everything on each compute node, so it takes a few minutes.

### Specify Experiment Parameters in JSON

The file save_iq_w_tx_file.json specifies all Shout parameters. Start with and edit a copy of save_iq_w_tx_file.json on your local machine.
            
Parameters:
- `txsamps`: from which file to transmit samples; created in "Create Transmitted Packet IQ File" above.
- `txrate` and `rxrate`: sampling rate at TX and RX.
- `txgain` and `rxgain`: TX and RX gain.
- `txfreq` and `rxfreq`: TX and RX carrier/center frequency. *Make sure both match!*
- `txclients` and `rxclients`: nodes for transmission and reception. Same as requested in the experiment under "CBAND X310 radios". *Make sure both match!*
- `rxrepeat`: number of repeated sample collection runs. I usually use 4, but it is fine to use 1.
- `sync`: whether to enable sync between TX and RX. "true" enables the use of the external (white rabbit) synchronization.
- `nsamps`: number of samples to be collected. I want to have this greater than twice as long as the packet length, to make sure I get a full packet no matter what.
- `wotxrepeat`: number of repeated sample collection runs without TX. 



### SSH into the orchestrator and clients
Once the experiemnt is ready, go to `List View` for the node hostnames.  Each host is listed with an SSH command for you. Mine says `ssh <username>@pc##-fort.emulab.net` in the row labelled `orch`.  This means that my username is `npatwari` and the orchestrator node is `pc##-fort.emulab.net`. There are five other `-comp` nodes, each with a hostname.  Those are my "radio hostnames".

Hope you have some screen space! We're going to need two terminals connected to the orchestrator, and one terminal for each radio hostname.

1. Use the following commands to start ssh and tmux sessions for the orchestor. Run the two lines separately, one per terminal.

    ```
    ssh -Y -p 22 -t <username>@<orch_node_hostname> 'cd /local/repository/bin && tmux new-session -A -s shout1 &&  exec $SHELL'
    ssh -Y -p 22 -t <username>@<orch_node_hostname> 'cd /local/repository/bin && tmux new-session -A -s shout2 &&  exec $SHELL'
    ```

[ssh -Y -p 22 -t npatwari@pc05-fort.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout1 &&  exec $SHELL']: #
[ssh -Y -p 22 -t npatwari@pc05-fort.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout2 &&  exec $SHELL']: #

3. Use the following command to start a ssh and tmux session for each of the clients:

    ```
    ssh -Y -p 22 -t <username>@<radio_hostname> 'cd /local/repository/bin && tmux new-session -A -s shout &&  exec $SHELL'
    ```
Note: `tmux` allows multiple remote sessions to remain active even when the SSH connection gets disconncted.

[ssh -Y -p 22 -t npatwari@pc10-fort.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout &&  exec $SHELL']: #
[ssh -Y -p 22 -t npatwari@pc11-fort.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout &&  exec $SHELL']: #
[ssh -Y -p 22 -t npatwari@pc07-fort.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout &&  exec $SHELL']: #

### Editing and Uploading Files

We need to upload to the compute nodes the files we created on our local machine: the 1) experiment parameters `save_iq_w_tx_file.json` and 2) transmitted signal file <TX-signal.iq>.

(Don't do this until the setup in "Instantiate an Experiment" is finished.)

I use the following commands to do this:

   `scp -r ./save_iq_w_tx_file.json <username>@pc##-fort.emulab.net:/local/repository/etc/cmdfiles/save_iq_w_tx_file.json`
   `scp -r ./QPSK_signal_2024-05-13.iq <username>@pc##-fort.emulab.net:/local/repository/shout/save_iq_w_tx_file.json`

where `pc##-fort.emulab.net` is the compute node name. Use whatever is on your list view of your compute nodes, in case they've got some other IP address.

### Transmission and reception 

1. Set up orchestrator-client connection

    (1) In one of your `orch` SSH sessions, run:
        ```
        ./1.start_orch.sh
        ```
    (2) In the SSH session for *each* of the clients, run:
        ```
        ./2.start_client.sh
        ``` 

2. Measurement collection
    In your other `orch` SSH session, run:

    ```
    ./3.run_cmd.sh
    ```

4. Wait for Shout to complete. 
5. Transfer the measurement file back to the local host. From your local terminal window:
   ```
   scp -r <username>@<orch_node_hostname>:/local/data/Shout_meas_<datestr>_<timestr> /<local_dir>
   ```
[scp -r npatwari@pc05-fort.emulab.net:/local/data/Shout_meas_05-13-2024_19-33-15 ./]: #
   
### Analyze the Received Signals

For the demonstration, we will analyze the received signals on Google Colab as our python notebook.  (You can also certainly run the python notebook locally on your own Jupyter Notebook if you have one installed on your computer.)   

Compress the collected measuremnt folder using zip in order to upload all its files to the Colab notebook.  Or if you want a previously collected dataset, there are some in this repo.

You will then load [our python notebook on Google Colab](https://colab.research.google.com/drive/1g2f8LmdU5wFYMR0MdZjbAmKMLLIxUWLe?usp=sharing).  You'll follow all of the instructions on this notebook.  That includes making a copy of the notebook (the linked file is read only); uploading the zipped measuremnt file; and picking the `txloc` and `rxloc` and `repNum` for the measurement you will analyze.  

### Student Assignment

I give an assignment to the students to write their own Matlab or Python code to operate the receiver. It is somewhat different than the receiver I provide above. I expect that in doing the lesson, students will learn about: 
- How to implement a MAP reception of QPSK-modulated signal
- How to use a preamble and sync for symbol time and phase synchronization
- How to plot eye diagrams, signal space project plots, and time and frequency-domain received signals.

The assignment is at: [Student_OTA_QPSK_Assignment.md](Student_OTA_QPSK_Assignment.md).

