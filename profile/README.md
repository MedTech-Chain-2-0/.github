# MedTech Chain 2.0


## Overview of MedTech Chain 2.0

MedTech Chain 2.0, is the second version of MedTech Chain 1.0, which is a querying platform for authorized cybersecurity researchers to run statistical queries on medical device data. With the new version, a homomorphic encryption layer was added in order to protect the values of the data while still providing useful results to the researchers. With this change, new types of informative queries were added with the support of homomorphic encryption as well.

[Watch the video!](demo.mp4)

### Homomorphic Encryption
Homomorphic encryption allows performing operations on encrypting data without having to decrypyt them. While partially homomorphic encryption can perform only one operation without decrypting, fully homomorphic encryption can perform all mathmetical operations with the ciphertexts. The new version allows the admins of the system to choose which type of encryption will be used in the application.

For partial homomorphic encryption, the system allows the admins to generate new encryption keys while preserving the old keys and the data. How this can be done will be explained in the 'Using the Application' part. This key change functionality is yet not implemented for fully homomorphic encryption.

### New Queries
With the version 2 of MedTech Chain, five new queries were added for the cybersecurity researchers: unique count, sum, histogram, standard deviation, and linear regression. All queries are ran over the device data assets added between the choosen timestamps and the filters by the user. Explanations of each query is as follows:

| **Query Name**    | **Description**   | **Works with PHE** | **Works with FHE** |
|-------------------|-------------------|-------------------|---------------------|
| `Unique Count`      | Given a field, returns how many unique values of that field exist |  No|No|
| `Sum`  | Given an integer field, returns the addition of that field | Yes              |Yes|
| `Histogram`  | Given an integer/timestamp field and a bin size, shows a histogram of the values with the given bin size | No              |No|
| `Standard Deviation`  | Given an integer/timestamp field, returns the mean and standard deviation of that field | No              |Yes|
| `Linear Regression`  |Given an integer and a timestamp field, returns the linear regression of the two fields| No              |Yes|


## MedTech Chain 2.0 Structure

- [frontend](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/frontend): React web interface for users.
- [backend](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/backend): Node.js API for business logic and data access.
- [tools](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/tools): Scripts for building and simulating the HyperLedger. It runs the chaincode as a service.
- [encryption](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/encryption): Program that runs partially homomorphic encryption with Paillier scheme.
- [fhe](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/fhe): Program that runs fully homomorphic encryption with the BFV scheme.
- [ttp](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/ttp): The Trusted Third Party that holds the encryption/decryption keys.
- [chaincode](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/chaincode): Program ran by the peers on the HyperLedger.
- [hospital]https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/hospital: Code that simulates posting medical device data to the system.
- [protos](https://gitlab.ewi.tudelft.nl/cse2000-software-project/2024-2025/cluster-o/12b/protos): Sevice for building the java classes.

## How to Run the App

How to run each repository individually can be found within their own directory. This part will explain how to run and connect every component together. Before starting, make sure that Docker Desktop is installed and opened in the computer.


### Protos

The first step is to make sure that the Java classes which will be used by other repositories should be generated. Run the following command to do this:

```bash
cd ../protos
make javabindings
```

After this is done,  encryption codes should be built to be able to run them from the TTP or the chaincode.

### TTP

For installing Paillier, the binary can be directly copied to the TTP:

```bash
cd ../ttp
./scripts/copy-lib.sh
```
If BFV is not installed or used, go to 'application.properties' set 'bfv.cli.enabled' to false and comment out 'bfv.cli.binary=./binaries/bfv_io' :


For installing BFV, You need to build the C++ code first. Start with installing the dependencies.

```bash
cd ../fhe
sudo apt update
sudo apt install -y build-essential cmake git libomp-dev
```


Then, the OpenFHE library that creates the BFV scheme should be built. This can be ran only once, if no changes to the fhe code is made.

```bash
git clone https://github.com/openfheorg/openfhe-development.git
cd openfhe-development
mkdir build && cd build
cmake .. -DBUILD_SHARED=ON -DBUILD_STATIC=OFF \
         -DBUILD_EXAMPLES=OFF -DBUILD_UNITTESTS=OFF -DBUILD_BENCHMARKS=OFF
make -j$(nproc)
sudo make install   
sudo ldconfig
cd ../../  
```

#### TTP in Docker

When the TTP is ran as a docker container, the C++ code is automatically built by the container. Just copy the fhe code and build the container.

```bash
cd ttp
./scripts/copy-fhe.sh
docker-compose -p medtechchain-ttp up -d --build
```

#### TTP in IntelliJ

Use the run configuration `run-configs/TtpApplication` in IntelliJ to run. Now, the fhe must also be built.

```bash
cd ../fhe
mkdir build && cd build
cmake ..      
make -j$(nproc)
```

Now in build/ you can find the binaries for the BFV program. Go to the TTP to copy this code. 

```bash
cd ../../ttp
cp build/bfv_io  ../ttp/binaries/
```

Now, the TTP is ready for encryption. 

### Chaincode and Tools

It is time to build the HyperLedger fabric. First, the chaincode should have access to the correct Java classes.

```bash
cd ../chaincode
./scripts/copy-protos.sh
```

Then, fully homomorphic encryption binaries can be copied to chaincode for performing the homomorphic operations on chain:

```bash
./scripts/copy-fhe.sh
```

Now, go to tools repository to set up the Hyperledger. Note that you can use `--light` with these commands to setup the HyperLedger with a singular hopital peer.


```bash
cd ../tools/fabric
./infra-clean.sh
./infra-setup.sh
./infra-start.sh <--light>
```

If the chaincode will be deployed normally, run the command `./cc-deploy <--light> 1.0.0 1` to deploy the chaincode. Note that this version does not support BFV encryption. If changes are made to the chaincode, 'cc-deploy' command can be ran again by increasing the version and the sequence number (e.g. `./cc-deploy 1.0.1 2`). If the data in the blockchain should be restored, repeat the steps from the `./infra-clean.sh` step.

If the chaincode will be deployed as a service, which is the suggested version for BFV, run the command `./cc-deploy-external <--light> 1.0.0 1` to deploy the chaincode. In the command line, the chaincode package identifier in the following format will be printed: "medtechchain_1.0.0:X" where X is a series of 64 characters. Go to the "docker-compose.yaml" in the chaincode repository and for each container, set the CORE_CHAINCODE_ID_NAME to this package identifier. Open a separate terminal and run the following command:

```bash
cd chaincode
docker compose up --build
```

When the build is done and the message "Waits for server to be terminated" is seen, go to the terminal where the chaincode was deployed and press any button. The chaincode as a service is now deployed.

### Backend

Now it is time to run the backend. Start by copying the Java classes and build the containers.

```bash
cd ../../backend
./scripts/copy-protos.sh
export SMTP_PASSWORD="" && docker-compose --profile deps -p medtechchain-ums-be up -d --build
```

Now, the SpringBoot app for the backend is ready to be ran:

```bash
./gradlew bootRun
```
### Frontend

After the backend is ready, frontend should be ran. 

```bash
cd ../frontend
npm install # installs the dependencies
npm run dev
```

For logging in as an admin, use the following crediantials:

```text
username: "yavor"  
password: "yavor"
```

For logging in as a researcher, use the following crediantials:

```text
username: "researcher"
password: "yavor"
```

### Hospital

Note: For now, BFV only encrypts integer fields. If timestamps want to be encrypted, they should be scaled down from hospital to not exceed the limit of 536903681 in the calculations, which is the plaintext modulus of the BFV scheme.

Now the last step is to upload medical device data from the hospitals which will be used in queries. Before simulating the hospitals, the protos should be copied:

```bash
./scripts/copy-protos.sh
```
Use the light mode to run only one hospital instead of two.



#### Hospital in Docker

```bash
docker-compose --profile <default|light> -p medtechchain-hpt up -d --build 
```


#### Hospital in IntelliJ

Use the run configuration `run-configs/Application` in IntelliJ to run.


After uploading data, stop the hospital containers to be able to run the queries. You can do this through the Docker Desktop.


## License

All code is released under the MIT license:

MIT License

Copyright (c) 2025 Andac Durmaz, Yavor Pachedzhiev, Sinan Onen,
Ege Yarar, Ivan Markov, Roland Kromes, Zekeriya Erkin

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
