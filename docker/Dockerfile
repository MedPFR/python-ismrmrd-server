# ----- First stage to build ismrmrd and siemens_to_ismrmrd -----
FROM python:3.9-slim AS mrd_converter
ARG  DEBIAN_FRONTEND=noninteractive
#ENV  TZ=America/Chicago

RUN  apt-get update && apt-get install -y git cmake g++ libhdf5-dev libxml2-dev libxslt1-dev libboost-all-dev libfftw3-dev
RUN  mkdir -p /opt/code

# ISMRMRD library
RUN cd /opt/code && \
    git clone https://github.com/ismrmrd/ismrmrd.git && \
    cd ismrmrd && \
    mkdir build && \
    cd build && \
    cmake ../ && \
    make -j $(nproc) && \
    make install

# siemens_to_ismrmrd converter
RUN cd /opt/code && \
    git clone https://github.com/ismrmrd/siemens_to_ismrmrd.git && \
    cd siemens_to_ismrmrd && \
    mkdir build && \
    cd build && \
    cmake ../ && \
    make -j $(nproc) && \
    make install

# Create archive of ISMRMRD libraries (including symlinks) for second stage
# Tar could preserve the symlinks
RUN cd /usr/local/lib && tar -czvf libismrmrd.tar.gz libismrmrd*

# ----- Start another clean build without all of the build dependencies of siemens_to_ismrmrd -----
FROM python:3.9-slim

#LABEL org.opencontainers.image.description="Python MRD Image Reconstruction and Analysis Server"
#LABEL org.opencontainers.image.url="https://github.com/kspaceKelvin/python-ismrmrd-server"
#LABEL org.opencontainers.image.authors="Kelvin Chow (kelvin.chow@siemens-healthineers.com)"

# Copy ISMRMRD files from last stage
COPY --from=mrd_converter /usr/local/include/ismrmrd        /usr/local/include/ismrmrd/
COPY --from=mrd_converter /usr/local/share/ismrmrd          /usr/local/share/ismrmrd/
COPY --from=mrd_converter /usr/local/bin/ismrmrd*           /usr/local/bin/
COPY --from=mrd_converter /usr/local/lib/libismrmrd.tar.gz  /usr/local/lib/
RUN cd /usr/local/lib && tar -zxvf libismrmrd.tar.gz && rm libismrmrd.tar.gz && ldconfig

# Copy siemens_to_ismrmrd from last stage
COPY --from=mrd_converter /usr/local/bin/siemens_to_ismrmrd  /usr/local/bin/siemens_to_ismrmrd

# xslt and hdf5 are dependencies for siemens_to_ismrmrd
RUN apt-get update && apt-get install --no-install-recommends -y libxslt1.1 libhdf5-103 git
RUN mkdir -p /opt/code

# Add bart toolbox (fixed release 0.7), after make, only bart under the folder bart is important
# The libararies still need trial and error to save some space, so far around 150Mb, libfftw3-dev seems not important
# bart itself < 10Mb
RUN  apt-get install --no-install-recommends -y make gcc libfftw3-dev liblapacke-dev libpng-dev libopenblas-dev wget
RUN cd /opt/code &&\
    wget https://github.com/mrirecon/bart/archive/v0.7.00.tar.gz &&\
    tar xzvf v0.7.00.tar.gz &&\
    cd bart-0.7.00  &&\
    make && make install &&\
    cd ..  &&  rm v0.7.00.tar.gz  &&\
    echo "PATH=$PATH:/opt/code/:/src"   > /etc/environment
#    make install will make bart in the /usr/local/bin folder
# ENV PATH="/usr/local/bin/bart:${PATH}"

# Create pythonpath for single file packages (without requiring setup.py or __init__.py)
# though not useful in chroot image
RUN mkdir -p /opt/code/pythonpath && cp /opt/code/bart-0.7.00/python/* /opt/code/pythonpath/
ENV PYTHONPATH "${PYTHONPATH}:/opt/code/pythonpath"
#ENV TOOLBOX_PATH=/usr/local/bin/bart

RUN  cd /opt/code \
     && git clone https://github.com/ismrmrd/ismrmrd-python.git \
     && cd /opt/code/ismrmrd-python \
     && pip3 install --no-cache-dir .
#pip3 warnings, make take care in the future
RUN  cd /opt/code \
     && git clone https://github.com/ismrmrd/ismrmrd-python-tools.git \
     && cd /opt/code/ismrmrd-python-tools \
     && pip3 install --no-cache-dir .

# matplotlib is used by rgb.py and provides various visualization tools including colormaps
# pydicom is used by dicom2mrd.py to parse DICOM data
RUN pip3 install --no-cache-dir matplotlib pydicom

# copy the server code
# RUN  cd /opt/code \
#      && git clone -b bart https://github.com/MedPFR/python-ismrmrd-server.git
# Or direct the Host to the root of this folder
COPY .  /opt/code/python-ismrmrd-server

# Ensure startup scripts have Unix (LF) line endings, which may not be true
# if the git repo is cloned in Windows
#RUN apt-get install -y dos2unix \
#    && find /opt/code/python-ismrmrd-server -name *.sh | xargs dos2unix \
#    && apt-get remove dos2unix -y

# Ensure startup scripts are marked as executable, which may be lost if files
# are copied in Windows
RUN find /opt/code/python-ismrmrd-server -name *.sh -exec chmod +x {} \;

# Cleanup files not required after installation
RUN  apt-get remove git wget gcc make -y \
     && apt-get clean \
     && rm -rf /var/lib/apt/lists/* \
     && rm -rf /root/.cache/pip

CMD [ "python3", "/opt/code/python-ismrmrd-server/main.py", "-v", "-H=0.0.0.0", "-p=9002", "-l=/tmp/python-ismrmrd-server.log"]
