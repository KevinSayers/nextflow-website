---
title: 'A Docker and Nextflow Walkthrough'
date: 2016-06-10T10:26:40+10:00
draft: false
tags: ['docker','containerisation']
categories: ['Tutorial']
authors: ['Evan Floden']
heroSubHeading: 'Docker for dunces, Nextflow for nunces'
heroBackground: 'posts/green-abstract.png'
heroBackgroundOverlay: false
heroHeight: 500
heroDiagonal: false
heroDiagonalFill: false
---

*Below is a step-by-step guide for creating [Docker](http://www.docker.io) images for use with [Nextflow](http://www.nextflow.io) pipelines. This post was inspired by recent experiences and written with the hope that it may encourage others to join in the virtualization revolution.*

Modern science is built on collaboration. Recently I became involved with one such venture between several groups across Europe. The aim was to annotate long non-coding RNA (lncRNA) in farm animals and I agreed to help with the annotation based on RNA-Seq data. The basic procedure relies on mapping short read data from many different tissues to a genome, generating transcripts and then determining if they are likely to be lncRNA or protein coding genes.

During several successful 'hackathon' meetings the best approach was decided and implemented in a joint effort. I undertook the task of wrapping the procedure up into a Nextflow pipeline with a view to replicating the results across our different institutions and to allow the easy execution of the pipeline by researchers anywhere.

Creating the Nextflow pipeline ([here](http://www.github.com/cbcrg/lncrna-annotation-nf)) in itself was not a difficult task. My collaborators had documented their work well and were on hand if anything was not clear. However installing and keeping aligned all the pipeline dependencies across different the data centers was still a challenging task.

The pipeline is typical of many in bioinformatics, consisting of binary executions, BASH scripting, R, Perl, BioPerl and some custom Perl modules. We found the BioPerl modules in particular where very sensitive to the various versions in the *long* dependency tree. The solution was to turn to [Docker](https://www.docker.com/) containers.

I have taken this opportunity to document the process of developing the Docker side of a Nextflow + Docker pipeline in a step-by-step manner.

### Docker Installation

By far the most challenging issue is the installation of Docker. For local installations, the [process is relatively straight forward](https://docs.docker.com/engine/installation). However difficulties arise as computing moves to a cluster. Owing to security concerns, many HPC administrators have been reluctant to install Docker system-wide. This is changing and Docker developers have been responding to many of these concerns with [updates addressing these issues](https://blog.docker.com/2016/02/docker-engine-1-10-security/).

That being the case, local installations are usually perfectly fine for development. One of the golden rules in Nextflow development is to have a small test dataset that can run the full pipeline in minutes with few computational resources, ie can run on a laptop.

If you have Docker and Nextflow installed and you wish to view the working pipeline, you can perform the following commands to obtain everything you need and run the full lncrna annotation pipeline on a test dataset.

{{< highlight bash >}}

    $ docker pull cbcrg/lncrna_annotation
    $ nextflow run cbcrg/lncrna-annotation-nf -profile test

{{< / highlight >}}

[If the following does not work, there could be a problem with your Docker installation.]

The first command will download the required Docker image in your computer, while the second will launch Nextflow which automatically download the pipeline repository and
run it using the test data included with it.

### The Dockerfile

The `Dockerfile` contains all the instructions required by Docker to build the Docker image. It provides a transparent and consistent way to specify the base operating system and installation of all software, libraries and modules.

We begin by creating a file `Dockerfile` in the Nextflow project directory. The Dockerfile begins with:

{{< highlight bash "linenos=table" >}}

    # Set the base image to debian jessie
    FROM debian:jessie

    # File Author / Maintainer
    MAINTAINER Evan Floden <evanfloden@gmail.com>

{{< / highlight >}}

This sets the base distribution for our Docker image to be Debian v8.4, a lightweight Linux distribution that is ideally suited for the task. We must also specify the maintainer of the Docker image.

Next we update the repository sources and install some essential tools such as `wget` and `perl`.

{{< highlight bash "linenos=table" >}}

    # Set the base image to debian jessie
    FROM debian:jessie

    # File Author / Maintainer
    MAINTAINER Evan Floden <evanfloden@gmail.com>

    RUN apt-get update && apt-get install --yes --no-install-recommends \
        wget \
        locales \
        vim-tiny \
        git \
        cmake \
        build-essential \
        gcc-multilib \
        perl \
        python ...

{{< / highlight >}}

Notice that we use the command `RUN` before each line. The `RUN` instruction executes commands as if they are performed from the Linux shell.

Also is good practice to group as many as possible commands in the same `RUN` statement. This reduces the size of the final Docker image. See [here](https://blog.replicated.com/2016/02/05/refactoring-a-dockerfile-for-image-size/) for these details and [here](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) for more best practices.

Next we can specify the install of the required perl modules using [cpan minus](http://search.cpan.org/~miyagawa/Menlo-1.9003/script/cpanm-menlo):

{{< highlight bash "linenos=table" >}}

    # Install perl modules
    RUN cpanm --force CPAN::Meta \
        YAML \
        Digest::SHA \
        Module::Build \
        Data::Stag \
        Config::Simple \
        Statistics::Lite ...

{{< / highlight >}}

We can give the instructions to download and install software from GitHub using:

{{< highlight bash "linenos=table" >}}

    # Install Star Mapper
    RUN wget -qO- https://github.com/alexdobin/STAR/archive/2.5.2a.tar.gz | tar -xz \
        && cd STAR-2.5.2a \
        && make STAR

{{< / highlight >}}

We can add custom Perl modules and specify environmental variables such as `PERL5LIB` as below:

{{< highlight bash "linenos=table" >}}

    # Install FEELnc
    RUN wget -q https://github.com/tderrien/FEELnc/archive/a6146996e06f8a206a0ae6fd59f8ca635c7d9467.zip \
        && unzip a6146996e06f8a206a0ae6fd59f8ca635c7d9467.zip \
        && mv FEELnc-a6146996e06f8a206a0ae6fd59f8ca635c7d9467 /FEELnc \
        && rm a6146996e06f8a206a0ae6fd59f8ca635c7d9467.zip

    ENV FEELNCPATH /FEELnc
    ENV PERL5LIB $PERL5LIB:${FEELNCPATH}/lib/

{{< / highlight >}}

R and R libraries can be installed as follows:

{{< highlight bash "linenos=table" >}}

    # Install R
    RUN echo "deb http://cran.rstudio.com/bin/linux/debian jessie-cran3/" >>  /etc/apt/sources.list &&\
    apt-key adv --keyserver keys.gnupg.net --recv-key 381BA480 &&\
    apt-get update --fix-missing && \
    apt-get -y install r-base

    # Install R libraries
    RUN R -e 'install.packages("ROCR", repos="http://cloud.r-project.org/"); install.packages("randomForest",repos="http://cloud.r-project.org/")'

{{< / highlight >}}

For the complete working Dockerfile of this project see [here](https://github.com/cbcrg/lncRNA-Annotation-nf/blob/master/Dockerfile)

### Building the Docker Image

Once we start working on the Dockerfile, we can build it anytime using:

{{< highlight bash >}}

    $ docker build -t skptic/lncRNA_annotation .

{{< / highlight >}}

This builds the image from the Dockerfile and assigns a tag (i.e. a name) for the image. If there are no errors, the Docker image is now in you local Docker repository ready for use.

### Testing the Docker Image

We find it very helpful to test our images as we develop the Docker file. Once built, it is possible to launch the Docker image and test if the desired software was correctly installed. For example, we can test if FEELnc and its dependencies were successfully installed by running the following:

{{< highlight bash >}}

    docker run -ti lncrna_annotation

    cd FEELnc/test

    FEELnc_filter.pl -i transcript_chr38.gtf -a annotation_chr38.gtf \
    > -b transcript_biotype=protein_coding > candidate_lncRNA.gtf

    exit # remember to exit the Docker image

{{< / highlight >}}

### Tagging the Docker Image

Once you are confident your image is built correctly, you can tag it, allowing you to push it to [Dockerhub.io](https://hub.docker.com/). Dockerhub is an online repository for docker images which allows anyone to pull public images and run them.

You can view the images in your local repository with the `docker images` command and tag using `docker tag` with the image ID and the name.

{{< highlight bash >}}

    $ docker images

    REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
    lncrna_annotation                        latest              d8ec49cbe3ed        2 minutes ago       821.5 MB

    docker tag d8ec49cbe3ed cbcrg/lncrna_annotation:latest

{{< / highlight >}}

Now when we check our local images we can see the updated tag.

{{< highlight bash >}}

    $ docker images

    REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
    cbcrg/lncrna_annotation                 latest              d8ec49cbe3ed        2 minutes ago       821.5 MB

{{< / highlight >}}

### Pushing the Docker Image to Dockerhub

If you have not previously, sign up for a Dockerhub account [here](https://hub.docker.com/). From the command line, login to Dockerhub and push your image.

{{< highlight bash >}}

    $ docker login --username=cbcrg
    $ docker push cbcrg/lncrna_annotation

{{< / highlight >}}

You can test if you image has been correctly pushed and is publicly available by removing your local version using the IMAGE ID of the image and pulling the remote:

{{< highlight bash >}}

    $ docker rmi -f d8ec49cbe3ed

{{< / highlight >}}

    # Ensure the local version is not listed.

{{< highlight bash >}}

    $ docker images
    $ docker pull cbcrg/lncrna_annotation

{{< / highlight >}}

We are now almost ready to run our pipeline. The last step is to set up the Nextflow config.

### Nextflow Configuration

Within the `nextflow.config` file in the main project directory we can add the following line which links the Docker image to the Nexflow execution. The images can be:

* General (same docker image for all processes):

{{< highlight groovy >}}

        process {    
            container = 'cbcrg/lncrna_annotation'
        }

{{< / highlight >}}

* Specific to a profile (specified by `-profile crg` for example):

{{< highlight groovy >}}

        profile {
            crg {
                container = 'cbcrg/lncrna_annotation'
            }
        }

{{< / highlight >}}

* Specific to a given process within a pipeline:

{{< highlight groovy >}}

        $processName.container = 'cbcrg/lncrna_annotation'

{{< / highlight >}}

In most cases it is easiest to use the same Docker image for all processes. One further thing to consider is the inclusion of the sha256 hash of the image in the container reference. I have [previously written about this](http://www.nextflow.io/blog/2016/best-practice-for-reproducibility.html), but briefly, including a hash ensures that not a single byte of the operating system or software is different.

{{< highlight groovy >}}

        process {    
            container = 'cbcrg/lncrna_annotation@sha256:9dfe233b...'
        }

{{< / highlight >}}

All that is left now to run the pipeline.

{{< highlight bash >}}

    $ nextflow run lncRNA-Annotation-nf -profile test

{{< / highlight >}}

Whilst I have explained this step-by-step process in a linear, consequential manner, in reality the development process is often more circular with changes in the Docker images reflecting changes in the pipeline.

### CircleCI and Nextflow

Now that you have a pipeline that successfully runs on a test dataset with Docker, a very useful step is to add a continuous development component to the pipeline. With this, whenever you push a modification of the pipeline to the GitHub repo, the test data set is run on the [CircleCI](http://www.circleci.com) servers (using Docker).

To include CircleCI in the Nexflow pipeline, create a file named `circle.yml` in the project directory. We add the following instructions to the file:

{{< highlight yaml >}}

    machine:
        java:
            version: oraclejdk8
        services:
            - docker

    dependencies:
        override:

    test:
        override:
            - docker pull cbcrg/lncrna_annotation
            - curl -fsSL get.nextflow.io | bash
            - ./nextflow run . -profile test

{{< / highlight >}}

Next you can sign up to CircleCI, linking your GitHub account.

Within the GitHub README.md you can add a badge with the following:

    ![CircleCI status](https://circleci.com/gh/cbcrg/lncRNA-Annotation-nf.png?style=shield)


### Tips and Tricks

**File permissions**: When a process is executed by a Docker container, the UNIX user running the process is not you. Therefore any files that are used as an input should have the appropriate file permissions. For example, I had to change the permissions of all the input data in the test data set with:

{{< highlight bash >}}

 	 $ find <data-path> -type f -exec chmod 644 {} \;
 	 $ find <data-path> -type d -exec chmod 755 {} \;

{{< / highlight >}}

### Summary
This was my first time building a Docker image and after a bit of trial-and-error the process was surprising straight forward. There is a wealth of information available for Docker and the almost seamless integration with Nextflow is fantastic. Our collaboration team is now looking forward to applying the pipeline to different datasets and publishing the work, knowing our results will be completely reproducible across any platform.
