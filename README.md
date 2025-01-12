# CompleteWGS
This is a pipline the enables the mapping, variant calling, and phasing of input fastq files from a PCR free and a Complete Genomics' stLFR library of the same sample. Running this pipeline results in a highly accurate and complete phased vcf. We recommend at least 40X depth for the PCR free library and 30X depth for the stLFR library. Below is a flow chart which summarizes the processes used in the pipeline. *Note, SV detection has not yet been enabled on this version of the pipeline.

![image](https://github.com/CGI-stLFR/CompleteWGS/assets/81321463/e73a2837-f60a-4a28-8d48-8eeb9e580905)

# Installation instructions: 
1. On a Linux server, install singularity >= 3.8.1 with root on every nodes.
   
2. Download the singularity images (internet connection required) by the following commands:
 
cat <<EOF > CWGS.def
Bootstrap: docker
From: stlfr/cwgs:1.0.3
%post
    cp /90-environment.sh /.singularity.d/env/
EOF
 
singularity build --fakeroot CWGS.sif CWGS.def

If the singularity doesn't support --fakeroot, you need sudo permission to run this command:
sudo singularity build CWGS.sif CWGS.def
 
singularity exec -B`pwd -P` CWGS.sif cp -rL /usr/local/bin/CWGS /usr/local/bin/runit /usr/local/app/CWGS/demo .

3. Download the database (internet connection required) by this command:
 
./CWGS -createdb
 
This command will download around 32G data from internet and build index locally, which will occupy another 30G storage.
If users are using MegaBolt or ZBolt nodes, they should create database by ./CWGS -createdb –megabolt .
 
4. Test demo data:
 
cat << EOF > samplelist.txt
sample  stlfr1                      stlfr2                      pcrfree1                 pcrfree2
demo    demo/stLFR_demo_1M_1.fq.gz  demo/stLFR_demo_1M_2.fq.gz  demo/PF_demo_1M_1.fq.gz demo/PF_demo_1M_2.fq.gz
EOF
 
./CWGS samplelist.txt -local
 
Test demo data on clusters by SGE (Sun Grid Engine):
 
./CWGS samplelist.txt --queue mgi.q --project none
 
Test demo data on clusters by SGE (Sun Grid Engine) with MegaBolt/ZBolt nodes:
 
./CWGS samplelist.txt --queue mgi.q --project none --use_megabolt true --boltq fpga.q

# Running the pipeline:
1. Generate sample.list.
   start from fastq files (default)
      E.g.
      cat << EOF > sample.list
      sample	stlfr1	stlfr2	pcrfree1	pcrfree2
      demo1	/path/to/stLFR_01_1.fq.gz	/path/to/stLFR_01_2.fq.gz	/path/to/PCRfree_01_1.fq.gz	/path/to/PCRfree_01_2.fq.gz
      demo2	/path/to/stLFR_02_1.fq.gz	/path/to/stLFR_02_2.fq.gz	/path/to/PCRfree_02_1.fq.gz	/path/to/PCRfree_02_2.fq.gz
      EOF
   
      *paths above can be both absolute and relative
   
    start from barcode split fastq files (set --skipBarcodeSplit true)
      format same as above.
   
    start from PCR-free and stLFR bam files (set --frombam true)
      E.g.
      cat << EOF > sample.list
      sample	stlfrbam	pfbam
      demo1	/path/to/stLFR_01.bam	/path/to/PCRfree_01.bam
      demo2	/path/to/stLFR_02.bam	/path/to/PCRfree_02.bam
      EOF
  3. Run settings
      Set CPU
          --cpu2 INT
            Specify cpu number for QC, markdup, bam downsample, merge bam, bam stats calculation. [24]

          --cpu3 INT
            Specify cpu number for alignment and short variants calling (including BQSR and VQSR, if specified). [48]

      Sample the input fastq files
          --sampleFq BOOL
            if you want to initially sample the fastq file, set it true. [false]

          the following settings are valid only sampleFq is true
          --stLFR_fq_cov INT [only valid when '--sampleFq true']
          sample stLFR reads to this coverage [40] 
          
          --PF_fq_cov INT [only valid when '--sampleBam true']
          sample PCR-free reads to this coverage [50] 

      Alignment and variant calling relevant settings
          --align_tool STRING
            Specify the alignment tool for stLFR reads. [lariat]
            Supports:
              bwa
              lariat
              bwa,lariat (this will execute both)

          --var_tool STRING
            Specify the variant calling tools for merged bam file. [dv]
            Supports:
              gatk
              dv (DeepVariant)
              gatk,dv (this will execute both)
          * If two alignment tools ("lariat,bwa") and two variant calling programs ("gatk,dv") are specified, four result sets will be generated.
 
          --gatk_version STRING [only valid when '--var_tool' contains "gatk"]
            Specify the GATK version. [v4]
            Supports:
              v3
              v4

          --run_bqsr BOOL [only valid when '--var_tool' contains "gatk"]
            Run Base Quality Score Recalibration (BQSR) of GATK. [true]

          --run_vqsr BOOL [only valid when '--var_tool' contains "gatk"]
            Run Variant Quality Score Recalibration (VQSR) of GATK. [true]

          --split_by_intervals BOOL [only valid when '--var_tool' contains "gatk" and '--use_megabolt' is false]
            Utilizes -L option for GATK haplotypecaller; split by chromosome. [true]

          --dv_version STRING [only valid when '--var_tool' contains "dv"]
            Specify the DeepVariant version. [v1.6]
            Supports: 
              v1.6
              v0.7
            Current MegaBOLT DeepVariant version is v0.7; therefore, if you specify this option to "v1.6", MegaBOLT will not be used even if '--use_megabolt' is true.

      Markdup
          --markdup STRING
            Specify the mark duplicates tool. [biobambam2]
            Supports: 
              biobambam2 (much faster than picard; recommended)
              picard
              gatk4 (MarkDuplicatesSpark)
              sambamba (not recommended)
      Downsample the bam file
          --sampleBam BOOL
            Whether downsample the stLFR bam and PCR-free bam. [true]

          --stLFR_sampling_cov INT [only valid when '--sampleBam true']
            Downsample stLFR bam to the specified coverage. [30]

          --PF_sampling_cov INT [only valid when '--sampleBam true']
            Downsample PCRFree bam to the specified coverage. [40]
      Merge the bam
          --PF_lt_stLFR_depth INT
            Extract the intersection regions from the sampled stLFR bam with depth greater than (>) this value and PCRFree bam with depth less equal than (<=) this value. [10]

      Enable resuming the running
          --keepFiles BOOL
            By default, useless intermediate files will be deleted during the analysis to save storage. If you want to resume the run, set it true. [false]

      Debug mode
          -debug
          By default, each process only keeps the output files. If you want to check the intermediate files within a process, use this flag.


  4. Executor and MegaBOLT setting, four combinations:
      1. clusters by SGE (Sun Grid Engine) (default)
          Ensure the clusters contain at least one MegaBOLT queue.
          Ensure that the SGE system is functioning and installed in the /opt/gridengine directory (if the installation directory is different, specified with -sge option). Confirm the working queue and project number, which can be specified using --queue, --boltq, and --project for regular queue, MegaBOLT queue, and project id, respectively. Use "--project none" if the system doesn't need a project id.
          E.g.
          perl CWGS.pl sample.list -sge /opt/sysoft/sge --queue mb.q --boltq fpga.q --project none > run.log 2>&1 &
      2. on clusters by SGE with none MegaBOLT nodes.
          Run with "-no_mb" option. 
          E.g.
          perl CWGS.pl sample.list -no_mb > run.log 2>&1 &
      3. locally on a MegaBOLT machine
          Run with "-local" option. 
          E.g.
          perl CWGS.pl sample.list -local > run.log 2>&1 &
      4. locally on a NONE MegaBOLT machine.
          Run with "-local -no_mb" option. 
          E.g.
          perl CWGS.pl sample.list -local -no_mb > run.log 2>&1 &  

       Note that the order of parameters matters: single dash parameters (-opt) should be placed before all double dash parameters (-–opt).
