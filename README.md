Voice Quality Analyzer
======================

This is a set of scripts that integrate Sevana AQuA software with
FreeSWITCH. They allow you to receive and record calls, compare the
received audio with the reference file, and store the results in the
database.

We assume that Sevana AQuA executable and the license file are located
at the following locations:

```
/opt/sevana/aqua-v
/opt/sevana/aqua-v.lic
```

The installation instructions are designed for Debian Linux, and can be
adapted to other OS distributions. The audio analysis scores are written
to local syslog and visible in `/var/log/user.log`, and to a local MySQL
database.

The FreeSWITCH dialplan is supposed to call the `run_proc` script with
the file name of the audio recording in WAV format. The file name is
expected to contain several important parts dividied by underscore: 1)
UNIX Epoch timestamp; 2) UUID of the call; 3) call source indicator
string. The source indicator is a string that allows you to identify how
this recording has been made. For example, it can contain the caller and
callee IDs. The source indicator is stored as-is in the database, and
shoudl not exceed 256 characters.

The script forks a child process and returns the control immediately to
FreeSWITCH, so that the channel can be removed from FreeSWITCH
memory. The forked children make sure that only one execution of AQuA
binary happens at a time.

The `FSQA_SOUNDFILES` SQL table contains the names of reference WAV
files. By default, the audio is compared against the 2-minute ITU-T
reference recording, and an alternative reference audio can be supplied
for a specific source indicator.


Installing the scripts and prerequisites
----------------------------------------


```
apt-get update && apt-get install -y git mysql-client mysql-server \
  libdbd-mysql-perl libuuid-perl

cd /opt
git clone https://github.com/voxserv/fsqa.git

mysqladmin create fsqa
mysql fsqa </opt/fsqa/fsqa_dbtables.sql 

mkdir -p \
  /var/spool/fsqa/audio_archive \
  /var/spool/fsqa/audio_rec \
  /var/spool/fsqa/temp

chown freeswitch:freeswitch \
  /var/spool/fsqa/audio_archive \
  /var/spool/fsqa/temp


cat >>/etc/fstab <<'EOT'
tmpfs /var/spool/fsqa/audio_rec tmpfs defaults,size=1g 0 0
EOT

mount /var/spool/fsqa/audio_rec

mkdir /opt/sound/
cd /opt/sound/
wget http://www.voxserv.ch/media/ITU-T_P_50_BRITISH_ENGLISH.wav
wget http://www.voxserv.ch/media/ITU-T_P_50_BRITISH_ENGLISH_30s.wav
```

Example dialplan
----------------

This repository contains `fsqa.xml` which can be copied as-is into
`/etc/freeswitch/dialplan` (FreeSWITCH dialplan directory if you
installed FreeSWITCH from precompiled packages):

```
cp /opt/fsqa/fsqa.xml /etc/freeswitch/dialplan/
```

The following call processing is defined in `fsqa.xml`:

* If the caller ID number is in E.164 format, the leading plus sign is removed.

* Extension `pstn_XXX` (here XXX is the DID number that receives the
  call from PSTN) plays the 2-minute reference file and records the
  inbound audio channel, and then executes the quality analysis
  script. The source indicator string is a combination of caller and
  callee ID's.

* Extension `pstnm_XXX` (muted) plays the silence stream and records the
  inbound audio, and sends it to the quality processing after the call
  ends.

* Extension `fsqa_YYY` (here YYY is an arbitrary string) sets YYY as the
  source indicator, plays back the reference audio, and sends the
  recording to quality analysis.

* Extension `fsqam_YYY` does the same as above, but the silence stream
  is played back.

* Extension `speaker` just plays the reference audio and hangs up.


So, normally you would define a PSTN gateway in `sip_profiles/` in your
FreeSWITCH configuration, and send the incoimg calls to `pstn_123456789`
extension (here 123456789 stands for your DID number) in `fsqa`
context. The easiest way to do this is to define a separate SIP profile
and bind it to a dedicated UDP port, so that all inbound calls are sent
to `fsqa` context.

This repository includes `sip_profiles--fsqa.xml` which defines a SIP
profile binding to port 5083 and setting the codec preferences to G711
only.

```
cd /etc/freeswitch
cp /opt/fsqa/sip_profiles--fsqa.xml sip_profiles/fsqa.xml
mkdir sip_profiles/fsqa

# use your own ITSP name and SIP credentials
cat >sip_profiles/fsqa/itsp_example.xml <<'EOT'
<include>
  <!-- DID: 123456789 -->
  <gateway name="example_itsp_123456789">
    <param name="username" value="123456789"/>
    <param name="proxy" value="sip.example.com"/>
    <param name="password" value="secret"/>
    <param name="extension" value="pstn_123456789"/>
    <param name="expire-seconds" value="120"/>
    <param name="register" value="true"/>
    <param name="register-transport" value="udp"/>
    <param name="retry-seconds" value="30"/>
    <param name="ping" value="30"/>
  </gateway>
</include>
EOT

systemctl restart freeswitch
```

