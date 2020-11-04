# Oracle Database to Amazon S3 Replication via Oracle Golden Gate for BigData
___
- O passo a passo, está levando em consideração do goldendate já configurado no lado ORACLE.

## Escopo
	Será Replicado do banco de dados ORACLE para Amazon s3 no formato JSON.
	Será replicado as tabelas abaixo.

Oracle | S3
------------ | -------------
PROTHEUS.PCG040 | PROTHEUS.PCG040


## Pre-Requisitos
	* Java SE
	* Amazon Java SDK

Configuração java
	Criar diretório e extrair o instalador do java.

	mkdir -p /u01/app/oracle/product/jdk
	tar xvfz /tmp/jdk-8u261-linux-x64.tar.gz -C /u01/app/oracle/product/jdk/


Configuração Drives java Amazon

	mkdir /amazon
	cd /amazon
	wget https://sdk-for-java.amazonwebservices.com/latest/aws-java-sdk.zip
	unzip aws-java-sdk.zip
	rm -f aws-java-sdk.zip


### Realizar instalação do goldengate for bigdata.

Link para baixar o GoldenGate for BigData.
	https://www.oracle.com/br/middleware/technologies/goldengate-downloads.html

Após baixar os arquivos necessários, seguir os passos abaixo para realizar instalação.

mkdir -p /home/oracle/ggd

cd /home/oracle/ggd
	unzip -oq /tmp/OGG_BigData_Linux_x64_19.1.0.0.5.zip
	tar -xvf OGG_BigData_Linux_x64_19.1.0.0.5.tar


## Definir variaveis de ambiente

criar um arquivo /home/oracle/.oggd_env com as variaveis do goldengate for bigdata
	export OGG_HOME=/home/oracle/ggd
	export JAVA_HOME=/u01/app/oracle/product/jdk/jdk1.8.0_261
	export PATH=$JAVA_HOME/bin:$PATH
	export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64/server:$LD_LIBRARY_PATH

criar um arquivo /home/oracle/.ogg_env com as variaveis do goldengate for oracle database
	export OGG_HOME=/u01/app/oracle/product/gg
	export JAVA_HOME=/u01/app/oracle/product/jdk/jdk1.8.0_261
	export PATH=$JAVA_HOME/bin:$PATH
	export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64/server:$LD_LIBRARY_PATH

=========

Carregar as variaveis . /home/oracle/.oggd_env

Após as configurações acima, conectar no ggsci e criar os diretórios necessários.

./ggsci	
	
	GGSCI (srvdb-hml01) 1> CREATE SUBDIRS

	Creating subdirectories under current directory /home/oracle/ggd

	Parameter file                 /home/oracle/ggd/dirprm: created.
	Report file                    /home/oracle/ggd/dirrpt: created.
	Checkpoint file                /home/oracle/ggd/dirchk: created.
	Process status files           /home/oracle/ggd/dirpcs: created.
	SQL script files               /home/oracle/ggd/dirsql: created.
	Database definitions files     /home/oracle/ggd/dirdef: created.
	Extract data files             /home/oracle/ggd/dirdat: created.
	Temporary files                /home/oracle/ggd/dirtmp: created.
	Credential store files         /home/oracle/ggd/dircrd: created.
	Masterkey wallet files         /home/oracle/ggd/dirwlt: created.
	Dump files                     /home/oracle/ggd/dirdmp: created.

edit param mgr

	PORT 7909
	DYNAMICPORTLIST 7910-7935
	ACCESSRULE, PROG *, IPADDR *, ALLOW
	AUTORESTART ER *, RETRIES 6, WAITMINUTES 2, RESETMINUTES 30
	LAGCRITICALSECONDS 30
	LAGREPORTMINUTES 5


start mgr
	info all

	Program     Status      Group       Lag at Chkpt  Time Since Chkpt

	MANAGER     RUNNING


No Oracle Goldendate Database:
=========

Carregar as variaveis . /home/oracle/.ogg_env
	cd $OGG_HOME

./ggsci	
	
	edit param mgr

	PORT 7809
	DYNAMICPORTLIST 7810-7835
	ACCESSRULE, PROG *, IPADDR *, ALLOW
	AUTORESTART ER *, RETRIES 6, WAITMINUTES 2, RESETMINUTES 30
	LAGCRITICALSECONDS 30
	LAGREPORTMINUTES 5
	purgeoldextracts /home/oracle/ogg/dirdat/*, usecheckpoints, MINKEEPDAYS 2
	PURGEMARKERHISTORY MINKEEPDAYS 5, MAXKEEPDAYS 7, FREQUENCYHOURS 24
	PURGEDDLHISTORY MINKEEPDAYS 5, MAXKEEPDAYS 7, FREQUENCYHOURS 24

start mgr
	info all

	Program     Status      Group       Lag at Chkpt  Time Since Chkpt

	MANAGER     RUNNING


# Configuração de carga inicial Extract e Replicat

No OGG Oracle:
=========
Criar wallet para guardar acessos
	CREATE WALLET
	ADD CREDENTIALSTORE
	alter credentialstore add user ogg@DBOGG alias ogg_source

Parametros de extract para initial load

	GGSCI> ADD EXTRACT INITEX, SOURCEISTABLE

	GGSCI> EDIT PARAMS INITEX

	EXTRACT INITEX
	setenv (ORACLE_HOME="/u01/app/oracle/product/12.2.0.1/db_1")
	setenv (ORACLE_SID="DBOGG")
	SETENV (NLS_LANG = "AMERICAN_AMERICA.WE8MSWIN1252")
	USERIDALIAS ogg_source
	REPORTROLLOVER AT 05:30 ON friday
	RMTHOST 192.168.XXX.XX, MGRPORT 7909, COMPRESS
	RMTTASK REPLICAT, GROUP INITRE
	TABLE PROTHEUS.PCG040;


No OGG BDA:
=========
Parametros para replicat initial load
	
	GGSCI> ADD REPLICAT INITRE, SPECIALRUN

	GGSCI> EDIT PARAMS INITRE

	REPLICAT INITRE
	SETENV(GGS_JAVAUSEREXIT_CONF = 'dirprm/rsnow.properties')
	getEnv (JAVA_HOME)
	SETENV(LD_LIBRARY_PATH ='/u01/app/oracle/product/jdk/jdk1.8.0_261/jre/lib/amd64/server/:/u01/app/oracle/product/12.2.0.1/db_1/jdk/jre/lib/amd64/server/:/home/oracle/ggd')
	SETENV(GGHOME = '/home/oracle/ggd')
	TARGETDB LIBFILE libggjava.so SET property=dirprm/rsnow.properties

	GROUPTRANSOPS 1000
	MAP PROTHEUS.PCG040, TARGET *.*;



# Configuração de sincronização Extract e Replicat

No OGG Oracle:
==========
Parametros para o extract

	add extract esnow tranlog begin now
	add exttrail /home/oracle/ogg/dirdat/ss extract esnow

edit param esnow

	EXTRACT esnow
	setenv (ORACLE_HOME="/u01/app/oracle/product/12.2.0.1/db_1")
	setenv (ORACLE_SID="DBOGG")
	SETENV (NLS_LANG = "AMERICAN_AMERICA.WE8MSWIN1252")
	USERIDALIAS ogg_source
	REPORTROLLOVER AT 05:30 ON friday
	discardfile /u01/app/oracle/product/gg/dirrpt/esnow.dsc, purge
	exttrail /home/oracle/ogg/dirdat/ss
	TABLE PROTHEUS.PCG040;

Parametros para o pump
	Add extract dsnow, EXTTRAILSOURCE /home/oracle/ogg/dirdat/ss
	add rmttrail /home/oracle/ogg/dirdat/sf, extract dsnow

edit param dsnow

	EXTRACT dsnow
	setenv (ORACLE_HOME="/u01/app/oracle/product/12.2.0.1/db_1")
	setenv (ORACLE_SID="DBOGG")
	SETENV (NLS_LANG = "AMERICAN_AMERICA.WE8MSWIN1252")
	USERIDALIAS ogg_source
	RMTHOST 192.168.XXX.XX, MGRPORT 7909, COMPRESS
	RMTTRAIL /home/oracle/ogg/dirdat/sf
	TABLE PROTHEUS.PCG040;



No OGG BDA:
=========
Parametros para o replicat

	add replicat rsnow  exttrail /home/oracle/ogg/dirdat/sf

	edit param rsnow

	REPLICAT rsnow
	SETENV(GGS_JAVAUSEREXIT_CONF = 'dirprm/rsnow.properties')
	getEnv (JAVA_HOME)
	SETENV(LD_LIBRARY_PATH ='/u01/app/oracle/product/jdk/jdk1.8.0_261/jre/lib/amd64/server/:/u01/app/oracle/product/12.2.0.1/db_1/jdk/jre/lib/amd64/server/:/home/oracle/ggd')
	SETENV(GGHOME = '/home/oracle/ggd')
	TARGETDB LIBFILE libggjava.so SET property=dirprm/rsnow.properties

	GROUPTRANSOPS 1000
	MAP PROTHEUS.PCG040, TARGET *.*;


Editar arquivo rsnow.properties com as suas informações de acesso.

vi dirprm/rsnow.properties


	gg.handlerlist=filewriter
	gg.handler.filewriter.type=filewriter
	gg.handler.filewriter.fileRollInterval=10s
	gg.handler.filewriter.fileNameMappingTemplate=${tableName}_${currentTimestamp}.json
	gg.handler.filewriter.pathMappingTemplate=ogg-load
	gg.handler.filewriter.stateFileDirectory=ogg-load-state
	gg.handler.filewriter.format=json
	gg.handler.filewriter.finalizeAction=rename
	gg.handler.filewriter.fileRenameMappingTemplate=${tableName}_${currentTimestamp}.json
	gg.handler.filewriter.eventHandler=s3
	goldengate.userexit.writers=javawriter

	gg.eventhandler.s3.type=s3
	gg.eventhandler.s3.region=us-east-1
	gg.eventhandler.s3.bucketMappingTemplate=goldengate-protheus
	gg.eventhandler.s3.pathMappingTemplate=${tableName}_${currentTimestamp}
	gg.classpath=/home/oracle/ggd/dirprm/:/amazon/aws-java-sdk-1.11.890/lib/aws-java-sdk-1.11.890.jar:/amazon/aws-java-sdk-1.11.890/lib/*:/amazon/aws-java-sdk-1.11.890/third-party/lib/*:/amazon/aws-java-sdk-1.11.890/third-party/lib/jackson-annotations-2.6.0.jar
	gg.log=log4j
	gg.log.level=DEBUG
	javawriter.bootoptions=-Xmx512m -Xms32m -Djava.class.path=.:ggjava/ggjava.jar -Daws.accessKeyId=SUA_KEY -Daws.secretKey=SUA_SECRET


# Iniciar os processos de replicação.

No OGG Oracle:
==========
	GGSCI> start ESNOW
	GGSCI> start DSNOW

Iniciar os processos de sync primeiro e depois o processo de carga inicial.

	GGSCI> start INITEX

No OGG BDA:
=============
	GGSCI> start RSNOW


Acompanhar informações intial load
	
	GGSCI> INFO INITEX