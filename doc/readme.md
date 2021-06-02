### brief intro
#index
build docker image
build chroot image
run the docker image
test the recon algorithm
install ismrmrdviewer

docker related: image, a subsystem, contains everything, but only readable; container, running this image with a writable layer, although all changes won't change image after the container is done.
but the scanner machine uses chroot, (CHangeROOT), so export the image from docker, extract it into chroot image. Then some settings from docker won't pass on into chroot.
Even though, the docker itself is easier to test and run, so debug could be done on docker or even in local computer, and then transfer into the docker, then chroot.

#based on python-ismrmrd-server github project
#1, all these three start_server.sh, line 7 added, PYTHONPATH
#2, not yet TOOLBOX_PATH, not important
#3, if using bart, then line11 for now should be commented out, otherwise bart cannot work properly
#4, Dockerfile, add bart, after run 'make', run 'make install', then the created bart will be copied into the /usr/local/bin, which is the default PATH of chroot image.

### ---------build docker image----------
if every .py .sh script is done (although we could later modify them on the host)
change to base directory
```
docker build --no-cache -t firebart:1.0 -f ./docker/Dockerfile .
```
takes around 10 min, depends on computer, libraries
 -t, tag of the image, :xx is the version, default is latest, so better to name it when developing the image
 -f, which Dockerfile used for creating the docker image
 ., last directory, is the output, should be in a relatively local postion, otherwise docker will copy everything and upload and the compile, so here is the root directory of python-ismrmrd-server file.

### -------build chroot image-------------
after docker image, run doc2chr.sh (Docker to Chroot) for creating the chroot image
input: docker image tag, chroot image name, and the desired size of chroot image
```
bash /your/path/to/doc2chr.sh firebart:1.0(default) firebart1.img(default) 800(default 20% more spece)
```
### --------run the docker image---------------
#docker export image and import image
usually the first 3 character of the docker ID should be enough, but the image could also be called by tag or commit a new name
```
#if there is already the container
    docker commit containerID theirnewimage
    
#or create the container first with the existing docker image
    docker create --name theirnewimage imageID
```
export the container into a filesystem
```
docker save theirnewimage > /tmp/mynewimage.tar
```
then load the image into my docker image
```
docker load < /tmp/mynewimage.tar
docker images
```
#run the docker, Docker image: firebart:1.0
```
In Windows (create the C:\tmp first, or direct to another place):
    docker run -p=9002:9002 --rm -it -v C:\tmp:/tmp firebart:1.0

In MacOS/Linux:
    docker run -p=9002:9002 --rm -it -v /tmp:/tmp firebart:1.0
```
The server can be stopped by pressing ``ctrl-c``.
```
-p=9002:9002    Allows access to port 9002 inside the container from port 9002 on the host.  Change
                the first number to change the host port.
-it             Enables “interactive” mode with a pseudo-tty.  This is necessary for “ctrl-c” to
                stop the program.
--rm            Remove the container after it is stopped
-v /tmp:/tmp    Maps the /tmp folder on the host to /tmp inside the container.  Change the first
                path to change the host folder.  Log and debug files are stored in this folder.
```

In a separate command prompt, start another container of the Docker image:
```
docker run --rm -it -v /tmp:/tmp firebart:1.0 /bin/bash
```
#/bin/bash could also be bash, or other commands stored in /etc/profile
```
python3 /opt/code/python-ismrmrd-server/generate_cartesian_shepp_logan_dataset.py -o /tmp/phantom_raw.h5
```
Run the client and send data to the server in the other Docker container, via docker host default ip 172.17.0.1:
```
python3 /opt/code/python-ismrmrd-server/client.py -a 172.17.0.1 -p 9002 -G "dataset" -o /tmp/phantom_img.h5 /tmp/phantom_raw.h5
```
commands, more in the file client.py
```
-p 9002        Send data to port 9002.  This is sent to the host, which is then redirected to the
               server container due to the “-p” port mapping when starting the server. 
-o             Specifies the output file name
-G             Specifies the group name in the output file
```
then some useful commands from the next section
```
python3 /opt/code/python-ismrmrd-server/client.py -a 172.17.0.1 -p 9002 -G "dataset" -c simplefft -o /tmp/recon_img.h5 /tmp/phantom_raw.h5
```
to transfer the data between docker and the host, simply put files i.e. *.dat in the host directory /tmp as defined above
to check the reconstructed data from docker, simply start ismrmrdviewer on the host, open the files in this "shared" folder
### --------test the reconstruction scripts---------------
create a phantom
```
python3 generate_cartesian_shepp_logan_dataset.py -o phantom_raw.h5
python3 main.py -v
```
open another command prompt to send the h5 file for reconstruction
```
python3 client.py -c simplefft -o phantom_img.h5 phantom_raw.h5
```
-c, send the configuration to choose which .py script for reconstruction

using raw data from scanner, siemens_to_ismrmrd should be installed
```
siemens_to_ismrmrd -Z -f gre.dat -o gre_raw.h5
```
-f, directory of the input file
-o, directory of the output file
-Z, Convert all acquisitions from a multi-measurement (multi-RAID) file

using dicom data
```
python3 dicom2mrd.py -o dicom_img.h5 dicoms
```
dicoms is the folder containing dicom images

### ---------to view the reconstructed data (.h5 file), ismrmrdviewer is a nice choise-------
in a new python environment, better using conda to manage these environments
for example, miniconda, or anaconda
activate the base
```
conda activate
```
create an environment named ismrmrd
```
conda create --name ismrmrd
conda activate ismrmrd
```
install pip under conda, suggested, although pip could also be installed using sudo apt, but then the packages are installed in the root directory, not in the conda env distributed directory
```
conda install pip
```
then install ismrmrdviewer, all dependencies will be cached and installed
```
pip install git+https://github.com/ismrmrd/ismrmrdviewer.git
```
here you go
```
ismrmrdviewer
```


### -------------not important----------------
for now, environment needs only two things to be taken care of, bart and python
	bart once installed, only bart in the source folder bart is important, so it could be moved to one of the default workdir folder, and the python folder to python path
	pathon path should be set by .sh in the cmd (last line of Dockfile)
for the /etc/environment, using the following
```
RUN echo "MKL_THREADING_LAYER=GNU" >> /etc/environment #append, > write
RUN echo "#/bin/bash \n/opt/code/python-ismrmrd-server/main.py -v -H=0.0.0.0 -p=9002 -l=/tmp/share/debug/python-ismrmrd-server.log" >> /usr/bin/start_server
```

#to test the chroot image
#find a mount point outside the home directory
	sudo mount -o loop $chroot.img /media/tmpmnt
sudo chroot /media/tmpmnt bash #or /bin/bash, many cmds available in etc
then ctl+d or type exit to exit the chroot
sudo umount /media/tmpmnt
#but it will be easier to test on docker image





