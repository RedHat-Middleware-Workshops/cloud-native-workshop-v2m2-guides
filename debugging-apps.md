## Lab2 - Debugging Applications

In this lab, you will debug the coolstore application using Java remote debugging and look into line-by-line code execution as the code runs on Quarkus.

####1. Enable Remote Debugging

---

Remote debugging is a useful debugging technique for application development which allows looking into the code that is being executed somewhere else on a different machine and execute the code line-by-line to help investigate bugs and issues. Remote debugging is part of  Java SE standard debugging architecture which you can learn more about it in [Java SE docs](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/architecture.html){:target="_blank"}.

Quarkus in development mode enables hot deployment with background compilation, which means that when you modify your Java files and/or your resource files and refresh your browser, these changes will automatically take effect. This works too for resource files like the configuration property file.

This will also listen for a debugger on port 5005. If your want to wait for the debugger to attach before running you can pass -Ddebug on the command line. If you don’t want the debugger at all you can use -Ddebug=false.

An easier approach would be to use the Quarkus maven plugin to enable remote debugging on the Inventory pod. It also forwards the default remote debugging port, 5005, from the Inventory pod to your workstation so simplify connectivity.

Enable remote debugging on Inventory by running the following inside the **inventory** directory in the CodeReady Workspaces Terminal window:

`mvn compile quarkus:dev`

> The default port for remoting debugging is **5005** but you can change the default port using `-Ddebug=port_num`

You are all set now to start debugging using the tools of you choice.

Do not wait for the command to return! The Quarkus maven plugin keeps the forwarded port open so that you can start debugging remotely.

![Quarkus Debug]({% image_path debug-che-quarkus.png %})

####2. Remote Debug with CodeReady Workspace

---

CodeReady Workspaces provides a convenience way to remotely connect to Java applications running inside containers and debug while following the code execution in the IDE.

From the **Run** menu, click on **Edit Debug Configurations...**.

![Remote Debug]({% image_path debug-che-debug-config-1.png %})

The window shows the debuggers available in CodeReady Workspace. Click on the plus sign near the
Java debugger.

![Remote Debug]({% image_path debug-che-debug-config-2.png %})

Configure the remote debugger and click on the **Save** button:

* Check **Connect to process on workspace machine**
* Port: **5005**

![Remote Debug]({% image_path debug-che-debug-config-3.png %}

You can now click on _Save_ then_ _Debug_ button to make CodeReady Workspaces connect to the Inventory service running on OpenShift.

You should see a confirmation that the remote debugger is successfully connected.

![Remote Debug]({% image_path debug-che-debug-config-4.png %}){:width="700px"}

Open _com.redhat.coolstore.InventoryResource_ and double-click on the editor sidebar on the line number of the first line of the _getAvailability()_
method to add a breakpoint to that line. A start appears near the line to show a breakpoint is set.

![Add Breakpoint]({% image_path debug-che-breakpoint.png %})

Note that you can use the the following icons to switch between debug and terminal windows.

![Icons]({% image_path debug-che-window-guide.png %})

Invoke the endpoint URL of the Inventory service using **curl** command in a separate Terminal:

`curl http://localhost:8080/services/inventory/165613 ; echo`

Switch back to the debug panel and notice that the code execution is paused at the breakpoint on _InventoryResource_ class.

![Icons]({% image_path debug-che-breakpoint-stop.png %})

Click on the _Step Over_ icon to execute one line and retrieve the inventory object for the given product id from the database.

![Step Over]({% image_path debug-che-step-over.png %}){:width="700px"}

The _Variables_ panel allows you to see (and change) local variables.

![Debug]({% image_path debug-che-breakpoint-values.png %})

This would allow you to see for example value of _itemId_ variable passed into the method during execution.

![Watch Variables]({% image_path debug-che-variables.png %})

Look at the **Variables** window. You can see the value of _itemId_ along with the Stream variables used to retrieve the inventory.

The product id(_165613_) is a unique value to retrieve certain data from the Inventory database. If you might have unexpected result or errors in development, this debugging feature will help you find the root cause quickly then you will eventually fix the issue.

####3. Resume Debug and Confirm the Result

Click on the _Resume_ icon to continue the code execution and then on the stop icon to end the debug session. When you swich to _Terminal-2_ window, you will see the result of **curl** command.

~~~shell
[{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256}]
~~~

> If you wait too long, the `curl` command may timeout and you won't get any output. You can repeat the process to see it again.

![Icons]({% image_path debug-che-window-result.png %})

> NOTE: Make sure to stop Quarkus development mode by closing the terminal with the running app.

#####Congratulations!