# Blue Green Deployment
Sample node js application to demo blue-green deployment in Docker Cloud
<h1>Introduction</h1>
FROM Gabriel Schenker - thx
<p>This post is part of my series about <a href="https://lostechies.com/gabrielschenker/2016/01/23/implementing-a-cicd-pipeline/">implementing a CI/CD pipeline</a>. Please refer to <a href="https://lostechies.com/gabrielschenker/2016/01/23/implementing-a-cicd-pipeline/">this</a> post for an introduction and a full table of content. In this post I want to demonstrate step by step how to create a simple Node JS application and run it as a stack in <a href="https://cloud.docker.com/">Docker Cloud</a>. This includes creating a repository in GitHub and linking it to a repository in Docker Cloud where we will use the auto build feature to create Docker images of the application. I will also show how to scale and load balance the application for high availability and even discuss how we can exercise blue-green deployment to achieve zero down time deployments.</p>
<h1>The Application</h1>
<p>To not over complicate things we create a simple node JS application consisting of a <code>package.json</code> file and a <code>server.js</code> file. The content of the <code>package.json</code> file is</p>
<p><script src="https://gist.github.com/223592c60c03d534fc21734edaddc346.js"></script><noscript>
<pre><code class="language-json json">{
  &quot;name&quot;: &quot;blue-green&quot;,
  &quot;version&quot;: &quot;1.0.0&quot;,
  &quot;description&quot;: &quot;Sample node js application to demo blue-green deployment in Docker cloud&quot;,
  &quot;main&quot;: &quot;server.js&quot;,
  &quot;scripts&quot;: {
    &quot;test&quot;: &quot;jasmine-node spec&quot;
  },
  &quot;author&quot;: &quot;Gabriel Schenker&quot;,
  &quot;license&quot;: &quot;ISC&quot;
}</code></pre>
<p></noscript></p>
<p>And the content of the <code>server.js</code> file looks like this</p>
<p><script src="https://gist.github.com/2c7822fb9390b2ac0f3e118151afebe2.js"></script><noscript>
<pre><code class="language-javascript javascript">var http = require('http');
var port = 80;

http.createServer(function(req, res){
  var hostname = process.env.HOSTNAME;
  res.writeHead(200, { 'Content-Type': 'text/HTML' });
  var name = process.env.NAME || 'World';
  var body = 
    '&lt;h1&gt;Hello '+name+'&lt;/h1&gt; \
     &lt;p&gt;This is a demo for &lt;em&gt;blue-green deployment&lt;/em&gt;&lt;/p&gt; \
     &lt;p&gt;from server: '+hostname+'&lt;/p&gt; \
     &lt;p&gt;Version: v1&lt;/p&gt;';
  res.end(body);
}).listen(port, function(){
  console.log('Server running at port ' + port);
});</code></pre>
<p></noscript></p>
<p>The <code>test</code> command is not important at this point. We will come back to it later down. We can now test this application locally by running</p>
<p><code>npm start</code></p>
<p>on the command line. If we navigate to localhost in the browser we should see something along this</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/Local1.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/Local1.png" alt="" title="Local" width="380" height="260" class="alignnone size-full wp-image-1344" /></a></p>
<p>Note that we are listening on port 80! This is important as we move along.</p>
<h1>The Dockerfile</h1>
<p>If we want to Docker-ize the application we need a Dockerfile. The content looks like this</p>
<p><script src="https://gist.github.com/070e194b42068f9686d8b3ae515416d9.js"></script><noscript>
<pre><code class="language-dockerfile dockerfile">FROM node
RUN mkdir -p /app
WORKDIR /app

# Install app dependencies
COPY package.json /app/
RUN npm install

COPY . /app

EXPOSE 80 

ENTRYPOINT [&quot;npm&quot;, &quot;start&quot;]</code></pre>
<p></noscript></p>
<p>Specifically note how we expose port 80. This will be important once we use a load balancer in front of the sample application in the cloud.</p>
<p>We can now build this Docker Image locally</p>
<pre><code>docker build -t blue-green .
</code></pre>
<p>and run an instance (container) of it</p>
<pre><code>docker run -dt --name bg -p 80:80 blue-green
</code></pre>
<p>If in the browser we now navigate to 192.168.99.100 (or whatever the IP address of your Docker host is) then we should see something like this <a href="https://lostechies.com/gabrielschenker/files/2016/04/Container-Local.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/Container-Local.png" alt="" title="Container-Local" width="405" height="267" class="alignnone size-full wp-image-1347" /></a></p>
<h1>Pushing the code to GitHub</h1>
<p>Push the code to a new GitHub repository. In my case I called it <code>BlueGreen</code>. The repo I use can be found here: <a href="https://github.com/browngeek666/bluegreen">https://github.com/browngeek666/bluegreen</a> I have added a simple <code>.gitignore</code> file which makes sure node modules are not pushed to GitHub. I also added a <code>Readme.md</code> for documentation purposes.</p>
<h1>Creating a Repository in Docker Cloud</h1>
<p>Create a new repository in Docker Cloud and link it to the above GitHub repository. Try to build the repo in Docker Cloud. It should take a minute or so and succeed.</p>
<h1>Create a Stack in Docker Cloud</h1>
<p>Create a new stack in Docker Cloud and name it appropriately. The content of the <code>Stackfile</code> should look like this</p>
<p><script src="https://gist.github.com/03079bab920ddcbd3344316980c9ff1c.js"></script><noscript>
<pre><code class="language- ">lb:
  image: 'dockercloud/haproxy:latest'
  links:
    - web
  ports:
    - '80:80'
  restart: always
  roles:
    - global
web:
  image: 'clearmeasure/blue-green:latest'
  deployment_strategy: high_availability
  restart: always
  target_num_containers: 3
</code></pre>
<p></noscript></p>
<p>Here we have a definition for the two services <code>lb</code> and <code>web</code>. The former one represents our load balancer and the latter our node JS application. The web service will be scaled to 3 instances and since we are using <strong>high availability</strong> strategy the instances will be put on different nodes of our cluster. We use self-healing in the sense as we require the service instances to <strong>restart automatically</strong> in case of failure. We also need to assign the role <code>global</code> to the load balancer such as that the service is able to query the Docker Cloud API.</p>
<h1>Running the Stack</h1>
<p>We can now deploy and run the stack we just defined. As soon as all services are up and running we can access the endpoint of the stack. Please use the <strong>Service endpoint</strong> and not the <strong>Container endpoint</strong>.</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/Endpoints2.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/Endpoints2.png" alt="" title="Endpoints" width="594" height="392" class="alignnone size-full wp-image-1354" /></a></p>
<p>If everything went as supposed we should then see something like this</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/Remote-container.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/Remote-container.png" alt="" title="Remote-container" width="445" height="231" class="alignnone size-full wp-image-1356" /></a></p>
<p>If we refresh the browser a few times we will see that the <strong>from server:</strong> changes between <code>web-1</code>, <code>web-2</code> and <code>web-3</code> which shows us that our node JS application is indeed load balanced.</p>
<h1>High Availability for the Load Balancer</h1>
<p>We can now also scale our load balancer such as that it is highly available. For that we should change our <code>Stackfile</code> to look like this</p>
<p><script src="https://gist.github.com/6bf56460976011414ff785dce20275e5.js"></script><noscript>
<pre><code class="language- ">lb:
  image: 'dockercloud/haproxy:latest'
  deployment_strategy: high_availability
  target_num_containers: 3
  links:
    - web-blue
  ports:
    - '80:80'
  restart: always
  roles:
    - global
web:
  image: 'clearmeasure/blue-green:latest'
  deployment_strategy: high_availability
  environment:
    - NAME=Gabriel
  restart: always
  target_num_containers: 3</code></pre>
<p></noscript></p>
<p>Note line 3 and 4 that have been added. The former guarantees us that the instances of the load balancer are distributed to different cluster nodes while the latter tells us how many load balancer instances we want to run. After we have redeployed the whole stack we can see in the Node dashboard that indeed every node has now 2 containers on it, one for the load balancer and one for the node application.</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/Node-dashboard.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/Node-dashboard.png" alt="" title="Node-dashboard" width="577" height="414" class="alignnone size-full wp-image-1359" /></a></p>
<h1>Blue-Green Deployment</h1>
<p>Now we want to try to realize zero-downtime or non destructive deployment. This is called <strong>blue-green deployment</strong>. It basically means that we install the new version of a service in parallel to the current existing one. Once the new version is ready we reroute all traffic to the new version and can then decommission the previous version of the service.</p>
<h2>Installing the Docker Cloud CLI</h2>
<p>First we need to install the Docker Cloud CLI as described <a href="https://docs.docker.com/docker-cloud/tutorials/installing-cli/">here</a>. With this CLI we can script everything in Docker Cloud. Please note that if you are on a Windows machine it is a bit more complicated than just typing <code>pip install docker-cloud</code>. We first need to have Python installed on our system. We can do that using Chocolatey.</p>
<p><code>choco install python2</code></p>
<p>Now create a folder where you want to install Docker Cloud CLI. Navigate to this folder. And then I had to use the full path to <code>pip.exe</code></p>
<p><code>c:\python27\scripts\pip.exe install docker-cloud</code></p>
<p>Now we need to add the path to docker-cloud to our path variable. Do this in the <strong>Advanced System Settings</strong>. Once this is done open a new (Bash) console and verify that everything is OK <a href="https://lostechies.com/gabrielschenker/files/2016/04/docker-cloud-cli.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/docker-cloud-cli.png" alt="" title="docker-cloud-cli" width="405" height="82" class="alignnone size-full wp-image-1366" /></a></p>
<h2>Tagging Images</h2>
<p>Now we want to start to work with tagged Docker images. For this we have to adjust our repository. If we want to have a tagged image generated then we need to either use branches or tags in GitHub. I prefer to use tags. Let&#8217;s say we have a version <code>v1</code> and a version <code>v2</code> we want to play with then we can just create tags in Git as follows</p>
<p><code>$ git tag -a v1 -m "Version 1"</code></p>
<p>This creates a tag <code>v1</code> with a description <code>Version 1</code> at the current commit. We can then push this tag to origin by using</p>
<p><code>$ git push origin v1</code></p>
<p>Now we can modify our repository in Docker Cloud. We can add <strong>image tags</strong> that refer to Git branches or tags (in our case it is tags)</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/tagged-images.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/tagged-images.png" alt="" title="tagged-images" width="970" height="789" class="alignnone size-full wp-image-1361" /></a></p>
<h1>The Stackfile</h1>
<p>If we want to introduce <strong>blue-green deployment</strong> (also called <strong>non-destructive deployment</strong>) we need to modify our <code>Stackfile</code> as follows</p>
<p><script src="https://gist.github.com/742589f74df630b3342470ddbbf94d3d.js"></script><noscript>
<pre><code class="language- ">lb:
  image: 'dockercloud/haproxy:latest'
  deployment_strategy: high_availability
  links:
    - web-blue
  ports:
    - '80:80'
  restart: always
  roles:
    - global
  target_num_containers: 3
web-blue:
  image: 'browngeek666/blue-green:v1'
  deployment_strategy: high_availability
  environment:
    - NAME=DevOps Foundation Class
  restart: always
  target_num_containers: 3
web-green:
  image: 'browngeek666/blue-green:v1'
  deployment_strategy: high_availability
  environment:
    - NAME=DevOps Foundation Class
  restart: always
  target_num_containers: 1</code></pre>
<p></noscript></p>
<p>Here we have instead of one service called <code>web</code> two services called <code>web-blue</code> and <code>web-green</code>. Only one of the two services is active at a given time (that is the load balancer is routing the traffic to it). At the beginning the load balancer is wired to <code>web-blue</code>. We are starting with both services being in version <code>v1</code>. We can then redeploy the whole stack and once it is up and running we can test it. We should see this</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/blue-v1.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/blue-v1.png" alt="" title="blue-v1" width="419" height="197" class="alignnone size-full wp-image-1363" /></a></p>
<h2>Upgrade green to v2</h2>
<p>Use this command to upgrade <code>web-green</code> to version <code>v2</code></p>
<p><code>$ docker-cloud service set --image browngeek666/blue-green:v2 --redeploy --sync web-green</code></p>
<p>The service will be automatically redeployed. Now we can scale up <code>web-green</code> to also use 3 instances</p>
<p><code>$ docker-cloud service scale web-green 3</code></p>
<p>While we are doing this the application still happily uses <code>v1</code> of our node js image.</p>
<h2>Switch blue to green</h2>
<p>Now it is time to switch from web-blue to <code>web-green</code> and reroute the load balancer. We can do this using this command</p>
<p><code>$ docker-cloud service set --link web-green:web-green lb</code></p>
<p>And then we need to redeploy the load balancer service using this command</p>
<p><code>$ docker-cloud service redeploy lb</code></p>
<p>And we&#8217;ll have this outcome</p>
<p><a href="https://lostechies.com/gabrielschenker/files/2016/04/green-v2.png"><img src="https://lostechies.com/gabrielschenker/files/2016/04/green-v2.png" alt="" title="green-v2" width="449" height="272" class="alignnone size-full wp-image-1364" /></a></p>
<h2>Rollback</h2>
<p>If something goes terribly wrong we can just rollback by linking the load balancer again with <code>web-blue</code> and redeploying it.</p>
<p><script src="https://gist.github.com/8643be79938770dd5b2c913dbf0f7baf.js"></script><noscript>
<pre><code class="language-shell shell">$ docker-cloud service set --link web-blue:web-blue lb
$ docker-cloud service redeploy lb</code></pre>
<p></noscript></p>
<h1>Summary</h1>
<p>In this post I have shown how Docker Cloud can be used to provide CI as well as scalability, high availability and blue-green deployment to us with a few simple operations. Everything can be automated through the <strong>docker-cloud CLI</strong>.</p>
