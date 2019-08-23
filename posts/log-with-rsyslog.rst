.. link:
.. description: Log centralized on remote server with RSyslog
.. tags: rsyslog
.. date: 2013/11/03 14:25:19
.. title: Log with RSyslog
.. slug: log-with-rsyslog

The logs on a system are the more useful files a developer or a sysadmin will use to know if everything is running in a proper way, or to find a bug in an application. For one or two servers, that's not very difficult to manage it, but when you've got more than ten servers to manage, go on each server can be really long. That's why rsyslog was created. Thanks to this program, you will be able to send all your logs (depending on your criteria) to a remote server which will catch them and allow you to collect all the logs from your servers.

There is a very simple design for rsyslog :

.. image:: /galleries/linux/rsyslog/rsyslog_server.png

As can imagine, this would be useful to have this on your cloud. And you can also have VMs from scratch or VMs created by an auto-scaling group. To not be search in every logs to find to which server these logs are, we also are going to create templates to separate each files in different directory according to the server's name. In this part we will log everything as files, but you can also store every logs in databases and use webapps to have a user friendly view of every logs.

So first go on the machine you want to use as log server. Here I'm using Debian 7, and the config is the same on RedHat. We're going to modify /etc/rsyslog.conf which is the main conf file of rsyslog.

So first, we will consider that the syslog server is a syslog client, we can also comment a part of the configuration, and at the same time, activate the listening system.

.. code::

   #  /etc/rsyslog.conf    Configuration file for rsyslog.
   #
   #                       For more information see
   #                       /usr/share/doc/rsyslog-doc/html/rsyslog_conf.html
   #################
   #### MODULES ####
   #################

   $ModLoad imuxsock # provides support for local system logging
   $ModLoad imklog   # provides kernel logging support
   #$ModLoad immark  # provides --MARK-- message capability

   # provides UDP syslog reception
   #$ModLoad imudp
   #$UDPServerRun 514
   # provides TCP syslog reception
   $ModLoad imtcp
   $InputTCPServerRun 514
   ###########################
   #### GLOBAL DIRECTIVES ####
   ###########################
   #
   # Use traditional timestamp format.
   # To enable high precision timestamps, comment out the following line.
   #
   $ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
   #
   # Set the default permissions for all log files.
   #
   $FileOwner root
   $FileGroup adm
   $FileCreateMode 0640
   $DirCreateMode 0755
   $Umask 0022
   #
   # Where to place spool and state files
   #
   $WorkDirectory /var/spool/rsyslog
   #
   # Include all config files in /etc/rsyslog.d/
   #
   $IncludeConfig /etc/rsyslog.d/*.conf
   ###############
   #### RULES ####
   ###############
   #
   # First some standard log files.  Log by facility.
   #
   auth,authpriv.*                        /var/log/auth.log
   #*.*;auth,authpriv.none         -/var/log/syslog
   #cron.*                         /var/log/cron.log
   daemon.*                       -/var/log/daemon.log
   kern.*                         -/var/log/kern.log
   lpr.*                          -/var/log/lpr.log
   mail.*                         -/var/log/mail.log
   user.*                         -/var/log/user.log
   #
   # Logging for the mail system.  Split it up so that
   # it is easy to write scripts to parse these files.
   #
   mail.info                      -/var/log/mail.info
   mail.warn                      -/var/log/mail.warn
   mail.err                       /var/log/mail.err
   #
   # Logging for INN news system.
   #
   news.crit                      /var/log/news/news.crit
   news.err                       /var/log/news/news.err
   news.notice                    -/var/log/news/news.notice
   #
   # Some "catch-all" log files.
   #
   #*.=debug;\
   #       auth,authpriv.none;\
   #       news.none;mail.none     -/var/log/debug
   #*.=info;*.=notice;*.=warn;\
   #       auth,authpriv.none;\
   #       cron,daemon.none;\
   #       mail,news.none          -/var/log/messages
   #
   # Emergencies are sent to everybody logged in.
   #
   *.emerg                                :omusrmsg:*

   #
   # I like to have messages displayed on the console, but only on a virtual
   # console I usually leave idle.
   #
   daemon,mail.*;\
   news.=crit;news.=err;news.=notice;\
   *.=debug;*.=info;\
   *.=notice;*.=warn       /dev/tty8
   # The named pipe /dev/xconsole is for the `xconsole' utility.  To use it,
   # you must invoke `xconsole' with the `-file' option:
   #
   #    $ xconsole -file /dev/xconsole [...]
   #
   # NOTE: adjust the list below, or you'll go crazy if you have a reasonably
   #      busy site..
   #
   #daemon.*;mail.*;\
   #       news.err;\
   #       *.=debug;*.=info;\
   #       *.=notice;*.=warn       |/dev/xconsole

