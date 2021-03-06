===================
Data Wallet
===================

************
Overview
************

An essential feature of the v3 of iExec is the ability to monetize all the aspects of a  computation: not only the hardware resources (Workers), but also the code and the data used in the computation (owned respectively by the dApp developer and the Data Owner). In this setting, the iExec software stack must guarantee the security of the code and data, and ensure no one can get access to them if not in the context of a blockchain-verified iExec transaction. Given the decentralized nature of the iExec platform, where computation can be performed by any anonymous user on unknown machines, the only way to fulfill these requirements is to take advantage of the recent developments in hardware-enforced secure execution. Consequently iExec v3 makes use of the Intel SGX technology to create a hardware-protected Trusted Execution Environment (TEE), where the code and data are out of reach and invisible even for a user with physical access to the execution machine.

Challenges/Requirements/Threat model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Data security is an important concern and a challenging problem even in the traditional, private and centralized computing environment. In an open and decentralized computing model like the iExec platform, these challenges are compounded several times. In addition to security against a traditional attack (eg a remote attacker exploiting a software/OS vulnerability to breach into a server and leak data), iExec must solve the following technical challenges:

* The owner of the machine where the computation takes place is anonymous and unvetted. We must assume he may steal any code or data available, especially if they are valuable.
* The data must be accessible publicly: since there can’t be any synchronisation/communication between a data owner and a data user, the (encrypted) dataset must be publicly available, on a third-party server or a decentralized file system like IPFS.
* Access must be granted in an asynchronous way: once again since there is no communication between a data owner and a data recipient/user, access to the data must be granted ahead-of-time by the data owner. In particular this prevents the use of re-encryption schemes, where the data would be encrypted specifically for the recipient using his public key. On the contrary in the iExec setting the symmetric key used for the data set/ application has to be available to the vetted application (ie whose end user has paid for the data, and is running authorized code) without supervision from the data owner.

Our solution is two-fold:

* Firstly, the iExec software supports the execution of tasks inside a trusted execution environment, using Intel SGX enclave technology. The code inside the enclave is guaranteed to behave as intended. It is responsible for retrieving the data and loading the right application to process. The code and the data inside an enclave are completely out-of-reach from code outside of the enclave - even for simple read access.

                                                   .. image:: ./_images/SGX.jpg

* Second we provide an auditable (open source) `Secret Management Service <https://github.com/iExecBlockchainComputing/SMS>`_ (SMS) software, whose purpose is to securely store the keys for the different actors involved (dataset owner, computation beneficiary). This solves the problem of delegating access rights: in essence the user delegates managing access to his data to the SMS. Of course the SMS has the same security challenges as the Worker (we must ensure it runs as expected, doesn’t spill the secrets, and is not spoofed by the owner of the machine on which it runs) hence **it must also run inside an SGX enclave** . As a company iExec will run its own version of the SMS, but the most security-minded users may very simply run their own SMS. Note that it could also be run by any willing third-party. The data owners will decide which SMS they entrust with their keys.

Security guarantees
~~~~~~~~~~~~~~~~~~~~

#. Private code and data will be symmetrically encrypted whenever publicly available (IPFS, public repository). Decryption will only take place inside an attested SGX enclave, whose code has been authorized by the data owner.
#. Private code and data will be encrypted client-side, on the machine of their owner, then pushed on a public data repository. The corresponding keys will only be communicated over TLS channels established with auditable programs, running in attested enclave.
#. The Data owner may limit the use of their data to a set of applications. In this case the key will only be sent to an attested enclave whose MREnclave matches the one whitelisted by the Data Owner.
#. The Data Owner may limit the use of their data to a reduced number of end users (beneficiaries). In this case, the result of the computation will be encrypted by the Worker and only the beneficiary will be able to decrypt it.
#. At all times, keys to the data will only be accessible by:

	* The iExec SMS service (open-source iExec code, attested by Intel IAS)
	* The iExec Dapp (with an MREnclave either guaranteed by its author or by an independent auditor) white-listed by the Data provider.
	* The enclave-based Scone software component, audited by a third-party and attested by Intel IAS.

****************
Implementation
****************

Across the iExec software stack there are two components that are mainly concerned with the security of data: the Worker and the SMS.
The Worker is made of two parts: the In-Enclave (trusted) Worker and the untrusted, standard Worker. All data-sensitive operations (ie handling the dataset keys and decryption, and subsequent processing of the data) are handled by the In-Enclave worker. The untrusted Worker handles interaction with the blockchain and the retribution process (PoCO) but it has no access to the data.
The Secret Management Service is an independent service, hosted either by the data provider or by a third-party, that implements a keystore in a secure, remotely attested enclave. It is mainly there to ensure that the keys are accessible at all times by an authorized user, ie that the computation can take place without an explicit authorization from the data provider.
The Worker and the SMS both run in an SGX enclave; they are managed by an instance of the Palaemon software running in its own enclave. The Palaemont software is responsible for creating the enclave, initializing it with the iExec software and attesting it locally.

Protocol:
~~~~~~~~~~
Here are the steps of an end-to-end encrypted execution:

#. The Data Owner symmetrically encrypts his dataset and pushes the result on an online repository (IPFS). He then assigns a unique (Ethereum) address to this data, by creating a dedicated smart contract for it.
#. The Data Owner establishes a secure connection with the SMS of his choice. The connection is established after verifying the SMS enclave certificate that proves to the Data Owner that the other endpoint of the TLS channel is an SGX enclave running the proper SMS software.
#. The Data Owner then pushes the symmetric key and address of the encrypted data, authenticated (signed) with his Ethereum private address, to the SMS enclave (over TLS). The SMS keeps a mapping {data key ⇔ data address}.
#. After being assigned a computation, the Worker receives the information on the task  which includes which app to run and which dataset to use (by reading them on the blockchain). With the address of the dataset he is able to retrieve which SMS is holding the corresponding symmetric key.
#. The Worker then asks the Palaemon service to create an enclave, load it with the downloaded app and attest it. Palaemon delivers a TLS certificate to the enclave, authenticated with the MREnclave (ie attesting the enclave is running the intended app).
#. The Worker downloads the encrypted data and place it in a special directory available reachable by the enclave Worker.
#. The enclave Worker then requests the data key to the SMS. The request is signed with the enclave private key; the request also contains the enclave certificate delivered by Palaemon (which links the public key corresponding to the enclave private key to the MREnclave of the enclave Worker). When receiving this request, the SMS retrieves the MREnclave vetted by the data owner (by reading it on the blockchain).
#. The SMS then verifies the certificate, and compare the MREnclave of the certificate with the one on the blockchain corresponding to the Dataset address. If the comparison is positive, it sends the data key (over the TLS channel)

Analysis:
~~~~~~~~~~

Since the data is encrypted by the data owner, their confidentiality and integrity depends on two conditions:

#. The keys to the data should not leak
#. The entity that may know the key and thus decrypt the data should not leak them

* 1. is ensured on one hand by transmitting the keys over TLS, and on the other hand by the design of the SMS, which transmits the keys only to an enclave with the right MREnclave (and thus the right application), and only for a computation whose beneficiary has been authorized by the Data Owner. By running the SMS inside an enclave and attesting it before sending it the keys, the Data Owner can make sure he is communicating with a proper SMS that will run as intended.
* 2. is ensured by remotely attesting the Worker enclave and auditing the code that runs inside it, making sure the code is not malicious and won’t leak the data.

************
Tutorial
************

In this tutorial we describe how to realize a fully secured computation with end-to-end data encryption using the iExec stack. As usual with the iExec platform, there are 4 different workflows, depending on your role in the transaction: dApp developer, data provider, hardware resource provider (worker) and computation requester.

Worker
~~~~~~~~~~

As a worker you need SGX-compatible hardware if you want to perform TEE-based computations. An unofficial list of SGX enabled CPU is available `here <https://github.com/ayeks/SGX-hardware>`_. Basically if your computer was built after 2016 it should be good.
In addition to an SGX-compatible CPU you also need to make sure the BIOS of your machine support the SGX extension. Most mainstream brand of computer (Dell, HP,...) do. If the SGX option is available in your BIOS then you need to enable it.
The next step is to install the drivers from Intel for the SGX extension. This can be done in one command line using the following script (on Ubuntu):

.. code-block:: bash

	curl -fssl https://raw.githubusercontent.com/SconeDocs/SH/master/install_sgx_driver.sh | bash

That’s it! Now you can register at your scheduler as an SGX compatible worker, and you’ll soon receive requests for SGX jobs.

Data provider
~~~~~~~~~~~~~~~~

If you want to protect your dataset you need to encrypt it before making it available on the iExec platform. There are two ways to encrypt your dataset, and only one of them is SGX compatible: see the `SDK tutorial <https://github.com/iExecBlockchainComputing/iexec-sdk/>`_ for more info. Make sure to encrypt your data with the right command.
In your working directory, put all your plaintext data in a /data-original folder and run the following command:

.. code-block:: bash

	docker run -v $PWD/data-original:/data-original  -v $PWD/data:/data -v $PWD/data-conf:/data-conf iexechub/sgx-data-encrypter

This image runs a script that encrypts your data, and writes the encryption key and corresponding hash on a file in data-conf named keytag. To enable the use of your data you need to upload this information in the SMS. This is\
done with the following SDK command:

.. code-block:: bash

	iexec tee push-secret --dataset /data-conf/keytag

Once this is done you need to create the contract for your dataset (more info `here <https://github.com/ayeks/SGX-hardware>`_)

.. code-block:: bash

	iexec dataset init
	iexec dataset deploy
	iexec order init
	iexec order publish --dataset

DApp developer
~~~~~~~~~~~~~~~~

Background
-----------
At its core the Intel SGX technology relies on the creation of special zones in memory called enclaves. Access to this zone is protected by the CPU, so that only code from inside the zone can access data in the enclave. If a code from outside the enclave - whatever its privilege level, even OS or hypervisor code -  tries to read a memory location that is part of the enclave the CPU will return an error.
The drawback is that whenever your program needs to use code outside the enclave - for example OS code  (eg system calls) for network or file system access - it needs to perform a special sequence of CPU instruction to leave the enclave securely. As a result to run a program natively you would need to rewrite it using Intel SDK and call these instructions manually, an impractical and potentially complex task.
To avoid this and make the use of SGX through iExec as developer friendly as possible, iExec provides a transparent integration with Scone, a runtime component developed by Scontain that allows to run applications in SGX enclaves in an unmodified way. We provide several docker images, that already include the Scone components as well as iExec integration code, that make the development of iExec-ready, SGX-enabled dApp as simple as a few Dockerfile lines.

Example: creating a Python 3 SGX dApp
---------------------------------------
**Step 1: Create the Dockerfile**

Your python scripts must be in a folder /app in your working directory. Create a Dockerfile named DockerFirstBuild from the following template. In particular you should install the python libraries your code will use with pip. Note that you don't need to build the docker image.

.. code-block:: bash

	FROM iexechub/python_scone:latest

	RUN pip install  attrdict

	COPY app	/app

	CMD python /app/myapp.py


**Step 2: Download and run our script**

To make your dApp compatible with iExec SGX workflow you need to precompute its MREnclave: this is basically a hash of your code, that will later enable us to check whether an enclave is actually running a genuine version of your code. You also need to authentify all your file: basically, we need to compute a hash of your image for later identification. This can be done in a single step using the following script - you just need to provide the name of your image as an argument.

.. code-block:: bash

	curl -fssl https://github.com/iExecBlockchainComputing/iexec-sdk/blob/master/README.md | bash

That’s it! Your app is now SGX compatible. Now you can deploy it using iExec SDK, following the normal dApp workflow (see `tutorial <https://docs.iex.ec/dockerapp.html#deploy-your-dapp>`_).

Computation requester/ beneficiary
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As a computation requester it is your choice to decide whether or not your execution should use iExec Data wallet. Note that most dataset available require the use of Trusted Execution to be decrypted, eg the dApp you use should be SGX compatible.

**Step 1: Create and push your encryption key**

One of the most interesting features of iExec Data wallet is the possibility to ask for your result to be encrypted inside the TEE: that is, only you will be able to read them. To allow this you need to generate a PKC key pair, and upload the public part to the SMS. This can be done in just one step with the iExec SDK:

.. code-block:: bash

	$ iexec tee generate-beneficiary-keys

Then you can push your public key to the SMS:

.. code-block:: bash

	$ iexec tee push-secrets


**Step 2: Order a E2E encrypted computation on iExec**

You can then follow the normal workflow to buy a computation as described in the `tutorial <https://docs.iex.ec/dockerapp.html#deploy-your-dapp>`_

.. code-block:: bash

	$ iexec order init

As in the normal iExec workflow, you should fill all the info needed in the iexec.json file (app, dataset, price, category). The only difference with the non-TEE workflow is in the tag:
In the iexec.json file, you should replace the tag by 0x0...01 (instead of 0x00...000):

.. code-block:: bash

	"requestorder": {
	"app": "0xAAdC3C643b79dbf8b761bA62283fF105930B20eb",
	"appmaxprice": 1500,
	"dataset": "0x570280a48EA01a466ea5a88d0f1C16C124BCDc3E",
	"datasetmaxprice": 12000,
	"workerpool": "0x0000000000000000000000000000000000000000",
	"workerpoolmaxprice": 5000,
	"volume": 1,
	"category": 3,
	"trust": 5,
	"tag": "0x0000000000000000000000000000000000000000000000000000000000000001",
	"beneficiary": "0xC08C3def622Af1476f2Db0E3CC8CcaeAd07BE3bB",
	"callback": "0x0000000000000000000000000000000000000000",
	"params": "{ \"0\": \"--help\" }",
	"requester": "0xC08C3def622Af1476f2Db0E3CC8CcaeAd07BE3bB"
	}

Then sign your orders, and publish your request order:

.. code-block:: bash

	$ iexec order sign
	$ iexec order publish --request

If your order is matched with the required components (app, dataset, worker), the computation will happen automatically, in a totally secure way.

**Step 3: Download and decrypt your results**

Once the computation is done you can download the result...

.. code-block:: bash

	$ iexec show task --download

...and decrypt it:

.. code-block:: bash

	$ iexec tee decrypt-results

And that's all! Your computation was executed in a protected enclave, and encrypted in-place: no one on Earth except you will be able to read the results.
