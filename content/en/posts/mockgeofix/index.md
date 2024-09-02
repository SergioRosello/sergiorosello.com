---
title: Leverage the power of open source and don't reinvent the wheel
date: '2018-12-01'
summary: 
---


<p>During this post, I’ll talk about how I needed a utility and how I cloned and modified a repository to achieve what I needed.</p>

<h2 id="introduction">Introduction</h2>

<p>One of the things I’m working on at <a href="https://waveapplication.com/">waveapplication</a> is test automation using calabash to ensure the quality of the apps before publishing to the app store. We are working towards a complete Continuous Integration and Continuous Delivery and to do so, we need end-to-end functional tests (Apart from Unit and Integration tests) In this specific case, I needed to fake GPS locations on a Android device.</p>

<h2 id="requirements">Requirements</h2>

<p>Given I’ll be needing this utility to setup the app to a specific state in order to check a specific feature/functionality of the app, I have specific requirements such as the ones below.</p>

<ul>
<li>Terminal-based utility</li>
<li>Lightweight</li>
<li>Ability to reproduce a route passed to it as a <em>.gpx</em> file.</li>
</ul>

<h2 id="research">Research</h2>

<p>Before starting to code frenetically, I stopped to do some research on the available methods and technologies available to accomplish given task. I found out that android’s <a href="https://developer.android.com/studio/command-line/adb?hl=es-419">ADB</a> provides developers with a specific command to change the devices location.</p>

<h3 id="geo-fix"><code>geo fix</code></h3>

<p>Given two arguments (<code>Longitude</code> and <code>Latitude</code>) in decimal degrees and optional parameters <code>satelites</code> and <code>altitude</code> it mocks the android device’s location to the parameters given.<br>
Once I knew the whole project could be done with <code>geo fix</code> I looked for open source repositories that had already implemented this functionality as I was sure of their existence. The first project I found was <a href="https://github.com/luv/mockgeofix">mockgeofix</a> created by <a href="https://github.com/luv">luv</a> this repository that had met all of my requirements except one, It required a app installed on the android device to mock the device’s location. This wasn’t an option for me but nevertheless I decided to clone the repository and give it a try.</p>

<h3 id="telnet">telnet</h3>

<p>This will be used to send the <code>geo-fix</code> commands to the android device given a port and an IP.</p>

<h2 id="adapting-mockgeofix-to-my-specific-needs">Adapting mockgeofix to my specific needs</h2>

<p>This utility had everything I needed to mock the device’s GPS location. It could even adapt the route to a given speed in km/h. The only thing incompatible with my requirements was the need of an external app for it to work but luckily, there is an alternative for that. These are the steps I followed to adapt the utility to my needs.</p>

<h3 id="clone-the-repository-and-remove-the-code-i-don-t-need">Clone the repository and remove the code I don’t need.</h3>

<p>Once I had gone through the code, I removed all of the java code (Used for the android application) and only kept the file I was interested in <code>run_sim.py</code> and all the files it depends upon. I also created a <code>requirements.txt</code> file to simplify the process needed to get the code up and running.</p>

<h3 id="add-includes-needed-for-telnet-and-arguments-for-telnet-security-verification">Add includes needed for telnet and arguments for telnet security verification</h3>

<p>To use Telnet, the program requires you to introduce a security password located in <code>~/.emulator_console_auth_token</code>. This grants permission to the client to execute methods such as <code>geo fix</code>.<br>
This means I had to include the argument option in the code.</p>

<h3 id="adapt-the-main-thread">Adapt the main thread</h3>

<p>This thread connected to the android app and then sent the <code>geo fix</code> commands to it. Now, it simply connects to the phone via telnet and sends the commands directly to it.</p>

<pre><code class="language-Python">def start_geofix(args):
    # Start the telnet connection
    tn = telnetlib.Telnet(args.ip, args.port)
    # Wait until keyword "OK" appears
    tn.read_until("OK")
    # Perform security verification
    # The code is located in ~/.emulator_console_auth_token
    tn.write("auth " + args.auth + "\n")
    tn.read_until("OK")
    while True:
        # Send "geo fix" commands forever
        tn.write("geo fix %f %f\r\n" % (curr_lon, curr_lat))
        time.sleep(UPDATE_INTERVAL)
</code></pre>

<h2 id="how-to-run-the-code">How to run the code:</h2>

<h3 id="requirements-1">Requirements</h3>

<ol>
<li><a href="https://www.python.org/downloads/release/python-2715/">python 2.7</a></li>
<li><a href="https://virtualenv.pypa.io/en/latest/">virtualenv</a></li>
</ol>

<h3 id="usage">Usage</h3>

<p>Note: To work, this needs a connected device or a emulated virtual device running</p>

<ol>
<li><code>git clone git@github.com:SergioRosello/mockgeofix.git</code></li>
<li>In project root directory: <code>source ENV/bin/activate</code></li>
<li>In project root directory: <code>pip install -r requirements.txt</code></li>
<li>In project root directory: <code>python run_sim.py -i targetIP -g path/to/gpx/file.gpx -t emulatorConsoleAuthToken</code></li>
</ol>

<h3 id="run-sym-py-s-parameters"><code>run_sym.py</code>’s parameters</h3>

<p>Below is a list of parameters used by <code>run\_sym.py</code> script.</p>

<h4 id="required-parameters">Required parameters</h4>

<ul>
<li><code>-i</code> or <code>--ip</code>: The ip address where it will find the device</li>
<li><code>-g</code> or <code>--gpx-file</code>: The <code>.gpx</code> file used to trace the route</li>
<li><code>-t</code> or <code>--auth</code>: Auth token for telnet authentication</li>
</ul>

<h4 id="not-required-parameters">Not required parameters</h4>

<ul>
<li><code>-p</code> or <code>-port</code>: The port used to connect. Default value: 5554</li>
<li><code>-S</code> or <code>--sleep</code>: Sleep between track points. Default value: 0.5</li>
<li><code>-s</code> or <code>--speed</code>: Speed in km/h (Takes precedence over -S)</li>
<li><code>-I</code> or <code>--listen-ip</code>: Run a HTTP server visualizing mocked locations on this ip</li>
<li><code>-P</code> or <code>--listen-port</code>: HTTP server’s port. Default value: 80</li>
</ul>

<h2 id="final-result">Final result</h2>

<p>The final state of the project can be seen <a href="https://github.com/SergioRosello/mockgeofix">here</a>. It leverages the power of telnet to communicate with the phone in order to remove the previously necessary application to send the <code>geo fix</code> command. Finally, this code accomplishes every requirement listed previously and I can use it to mock a route during the end-to-end functional tests.</p>

<h2 id="conclusion">Conclusion</h2>

<p>This project has been one of the first real open-source experiences I have had. This is what open source is about. Collaborating with developers worldwide to make things easier to one another. It’s really a beautiful thing to see and be a part of.</p>


