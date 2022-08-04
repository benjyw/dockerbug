FROM cimg/python:3.8

# hadolint ignore=DL3008,DL3004
RUN sudo apt-get update -y \
    && sudo apt-get install  --no-install-recommends -y python3-dev python3-distutils \
    amazon-ecr-credential-helper \
    awscli \
    build-essential \
    libssl-dev \
    libffi-dev \
    && sudo apt-get clean \
    && sudo rm -rf /var/lib/apt/lists/*

# hadolint ignore=DL3013
RUN pip install --no-cache-dir credstash

RUN echo "testing" >> test_file_parent1.txt
