# MLF-sample-project
A sample project for the Harvard ML Foundations group.

Right now this is a demonstration of how to use Singularity containers on the FASRC cluster. An important disclaimer: this is all new to me, so I am confident that some of these "best practices" are in fact suboptimal!

For your own project, you can create a new Singularity definition file based on `mlf-sample-project.def`, and of course modify the Python requirements listed in `requirements.txt`.

## Building the container

If the container for your project has already been built by you or someone else, congrats! You can skip to the next section. I've built a container for this sample project, which you can find at `/n/holystore01/LABS/barak_lab/Everyone/container_images/mlf-sample-project.sif`.

Now, back to business. There are two ways to build a container image from a definition file on the cluster. The first option is to use `proot`, which must be installed first:
```console 
curl -LO https://proot.gitlab.io/proot/bin/proot
chmod +x ./proot
```
This allows the container to be built without root access. To build an image based on the Singularity definition file, run the following command, replacing `/path/to/mlf-sample-project.sif` with the path to the image you want to create:
```console
$ singularity build /path/to/mlf-sample-project.sif mlf-sample-project.def
```
This will take on the order of 10 minutes to complete.
Note: `singularity` is not available from the login nodes, so the above command must be run from a compute or GPU node.

The only downside of the `proot` approach is that there are a few ways in which it doesn't perfectly emulate root access, so you could hypothetically run into permission problems when building if the definition file calls for certain forbidden actions.

The second option is to create a Sylabs account and build remotely on their free cloud service; instructions [here](https://github.com/fasrc/User_Codes/tree/master/Singularity_Containers#build-a-singularityce-container-remotely-from-singularity-definition-file-using-option---remote). This may (?) be slower than the proot option; idk, I haven't tried it. Also, the free Sylabs account limits you to performing a total of N builds for some constant N which I forget, and that's not cool.

## Using the container

To enter a shell in the container, run the following command:
```console
singularity shell --nv --cleanenv /path/to/mlf-sample-project.sif
```
Presto! You now are inside the container and can run commands to your heart's content.

The `--nv` flag is necessary if you want to use GPUs; otherwise the container's NVIDIA drivers will be tragically unused. The `--cleanenv` flag prevents the container from inheriting environment variables from the host machine. (Docker containers, unlike Singularity containers, have mutable file systems, so it is feasible to isolate the container from the host system. But with Singularity, isolationism is a lost cause; I'm including the `--cleanenv` flag here as a futile symbolic gesture.)

To run a shell command in the container (e.g. in a SLURM batch script), run the following command:
```console
singularity exec --nv --cleanenv /path/to/mlf-sample-project.sif COMMAND
```
For example, COMMAND might be a Python script.

## Important notes
- The container's file system is read-only. From within the container, you will by default have access to directories on the host's file system.
- One unfortunate consequence of this is that when you perform Python imports, the Python interpreter may search through Python packages in your home directory in addition to those installed inside the container (though the container's packages should be searched through first). This means that if you have a package installed in your home directory that is not installed in the container, Python may still be able to import it, leading to different behavior across users of the container. This can be avoided by always running python with the `-s` flag, which prevents it from searching through your home directory site-packages (and by ensuring that your `sys.path` doesn't have any other local paths). Unfortunately, Jupyter does not have an equivalent flag, so if you are using a Jupyter notebook, just be careful.
- JupyterLab or code-server will automatically store config files and data in your home directory, so these will not be specific to the individual container. (Storing them inside the container's file system is impossible because this is immutable.)

## JupyterLab
This container has JupyterLab installed by default. To run the server, run the following command from within the container:
```console
$ jupyter-lab --no-browser --port=8888 --ip='0.0.0.0'
```
And then, on your local machine, set up an SSH tunnel:
```console
$ ssh -NL 8888:holygpuXXXXXXX.rc.fas.harvard.edu:8888 USER@login.rc.fas.harvard.edu
```
with the appropriate values for `XXXXXXX` and `USER`.

## Code-server
Code-server is an off-brand variant of Visual Studio Code which runs entirely on the remote server — you interact with it through your browser. It is possible to use the standard Visual Studio Code desktop app and ssh into your Singularity containers, but the [solution](https://github.com/microsoft/vscode-remote-release/issues/3066#issuecomment-1019500216) requires your SSH config file to grow as (# cluster nodes) x (# containers), which is a pain. (So far, despite hours of wasted time and misleading conversations with GPT-4, I have failed to find a workable shortcut.)

This container has code-server installed by default. To get the code-server server going, run the following command from within the container:
```console
$ code-server --bind-addr 0.0.0.0:8888
```

To interact with the server on your local machine, you will need to set up an SSH tunnel locally:
```console
$ ssh -NL 8888:holygpuXXXXXXX.rc.fas.harvard.edu:8888 USER@login.rc.fas.harvard.edu
```
with the appropriate values for `XXXXXXX` and `USER`.

Then, you can access the server at `localhost:8888` in your browser. To be clear, the port 8888 is arbitrary; feel free to use your own favorite port (everyone has one).

By default, there is a password stored in `~/.config/code-server/config.yaml`.  You can remove the password requirement (what could possibly go wrong?...) with the following command: 
```console
$ sed -i.bak 's/auth: password/auth: none/' ~/.config/code-server/config.yaml
``` 

In Chrome, you can follow [these instructions](https://support.google.com/chrome/answer/9658361?hl=en&co=GENIE.Platform%3DDesktop) to install the code-server 'progressive web app', which will make it feel more like a native app and make you pine for the real Visual Studio Code less.)

### Extensions

Extensions can be installed as usual; note that they will be installed in your cluster home directory, not in the container's file system, because the container's file system is read-only. Because code-server is based on the open source version of VS Code, some extensions aren't listed by default. In particular, to install Github Copilot (which is the real reason I'm going through all this trouble instead of just using JupyterLab), you will need to download and install it manually:
```console
$ curl -L -o copilot.vsix.gz https://marketplace.visualstudio.com/_apis/public/gallery/publishers/GitHub/vsextensions/copilot/1.89.156/vspackage
$ gunzip copilot.vsix.gz
$ code-server --install-extension copilot.vsix
```
(To install the current version, go to the [VS Code marketplace page](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) and copy the download link.)