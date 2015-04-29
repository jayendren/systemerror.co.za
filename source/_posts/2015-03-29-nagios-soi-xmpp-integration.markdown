---
layout: post
title: "Nagios XMPP + SOI Integration"
date: 2015-04-29 20:26
comments: true
categories: ruby nagios soi
---
{% img center http://assets.nagios.com/images/header/Nagios.png %}
### About:

Ruby daemon to post nagios alerts to CA SOI.
Implementation spawned from Traiano's perl/xmpp nagbot
and performs xmpp posts to a chat room

### Requirements:
#### Ruby:

* Ruby187
* rubygems
* logger
* uri
* net/https
* facter

#### Perl:

* sendxmpp - http://www.djcbsoftware.nl/code/sendxmpp

#### Nagios:
* nagios server needs to be configured to write alerts to a named pipe/fifo in:
* nagios/etc/check_commands.cfg:

  `define command {  
       command_name notify-service-by-fifo 
       command_line /bin/echo "$HOSTNAME$ $NOTIFICATIONTYPE$ $SERVICEOUTPUT$ $SERVICEDESC$ $SERVICESTATE$ $LONGDATETIME$" > /var/spool/nagios/rw/ndbot.fifo
  }`

  `define command {  
       command_name                    notify-host-by-fifo
       command_line                    /bin/echo "$HOSTNAME$ $NOTIFICATIONTYPE$ $HOSTADDRESS$ $HOSTSTATE$ $HOSTOUTPUT$ $LONGDATETIME$" > /var/spool/nagios/rw/ndbot.fifo
  }`

* nagios/etc/nagios.cfg:  

  `global_host_event_handler=notify-host-by-fifo`
  `global_service_event_handler=notify-service-by-fifo`

* Create the fifo with:

   `mkfifo /var/spool/nagios/rw/ndbot.fifo`

#### FreeBSD:
* Copy of the rc_nagbot_soi script as:
  `/usr/local/etc/rc.d/nagbot_soi`
with executable permissions
* /etc/rc.conf entry:
  `nagbot_soi_enable="YES"`
* Copy of the ruby daemon (nagbot_soi.rb) as:
  `/usr/local/sbin/nagbot_soi.rb`

#### SOI:
* Example post (dev environment):
  `http://soi.lab/Soi/default.aspx?insid=win123001&sitmsg=cpu1&sev=4&alarm=54256`
* Definitions:
  * insid  : configuration item (maps to fqdn)
  * sitmsg : situation message (maps to alert message)
  * sev    : alert serverity
     * Normal   = 0
     * Minor    = 1
     * Major    = 2
     * Critical = 3
     * Down     = 4 
  * alarm  : silo alarm id (unique 5 digit integer)

### Implementation Workflow:

1. The daemon reads the named pipe and stores raw form of the data written to the pipe in the @alert variable, e.g:
  `ramc01.ops.lan  WARNING - load average: 8.39, 7.46, 5.81 server-NRPE-load WARNING Wed Jan 15 12:45:32 SAST 2014`

2. Matches OK|UP|WARN|CRIT|DOWN:
  * if @alert.match /OK|UP/ @state = 0 
  * if @alert.match /WARN/ @state  = 1
  * if @alert.match /CRIT/ @state  = 3
  * if @alert.match /DOWN/ @state  = 4

3. Converts the alert to an array (\s seperator), e.g:
  `["ramc01.ops.lan", "WARNING", "-", "load", "average:", "8.39,", "7.46,", "5.81", "server-NRPE-load", "WARNING", "Wed", "Jan", "15", "12:45:32", "SAST", "2014"]`

4. Extracts the following from the array:
  * @ci       = first element of @data array, e.g:
  `ramc01.ops.lan`
  * @sitmsg   = remaining elements from @data array to build the alert message, with URI encoding, e.g:
  `WARNING%20-%20load%20average:%208.39%20%207.46%20%205.81%20server-NRPE-load%20WARNING%20Wed%20Jan%2015%2012:45:32%20SAST%202014`
  * @alarm    = unique 5 digit number, needs to be identical when @state moves from 3 to 0 (critical to ok)
   * returns the date, e.g: 26 Feb 2014 = 26022104
   * returns the alert(data) in bytes
   * returns the data from the check in bytes
   * returns the alarm by adding the (date + alert) * check
    * e.g:
    * alert: remedy-ar01.ops.lan  PROCS WARNING: 7 processes with command name sendmail: check_remedy_arsystem_sendmail WARNING Wed Feb 26 15:42:22 SAST 2014
    * data : 114101109101100121459711448494611111211546109116110981171151051101011151154699111461229780827967838765827873787158551121141119910111511510111511910511610499111109109971101001109710910111510111010010997105108589910410199107951141011091011001219597114115121115116101109951151011101001099710510887658278737871871011007010198505449535852505850508365838450484952
    * check: 9511410110910110012195971141151211151161011099511510111010010997105108
    * alarm: 10852
    * alert: remedy-ar01.ops.lan  PROCS OK: 3 processes with command name sendmail: check_remedy_arsystem_sendmail OK Wed Feb 26 15:44:22 SAST 2014
    * data : 1141011091011001214597114484946111112115461091161109811711510511010111511546991114612297808279678379755851112114111991011151151011151191051161049911110910997110100110971091011151011101001099710510858991041019910795114101109101100121959711411512111511610110995115101110100109971051087975871011007010198505449535852525850508365838450484952
    * check: 9511410110910110012195971141151211151161011099511510111010010997105108
    * alarm: 10852

  * @soi_post = cummulation of the extracted data to post - e.g:
   `http://soi.lan/Soi/default.aspx?insid=ramc01.ops.lan&sitmsg=WARNING%20-%20load%20average:%208.39%20%207.46%20%205.81%20server-NRPE-load%20WARNING%20Wed%20Jan%2015%2012:45:32%20SAST%202014&sev=3&alarm=40363`

5. Posts the alert to the SOI server

6. Posts the alert to xmpp chat-room

### Global variables:
  
  @nagios_fifo = "/var/spool/nagios/rw/ndbot.fifo"         # path to nagios named pipe
  @log         = "/var/spool/nagios/nagbot_soi.log"        # path to logfile for this daemon
  @log_size    = 1024000                                   # logfile size in kb
  @logger      = Logger.new(@log, @log_size)               # logger instance
  @soi_prot    = "http"                                    # http or https
  @soi_host    = "soi.lan"                                 # ipv4/fqdn
  @soi_api     = "Soi/default.aspx?"                       # api 
  @soi_url     = "#{@soi_prot}://#{@soi_host}/#{@soi_api}" # soi url to post
  @soi_timeout = 600                                       # number of seconds after to wait before giving up
  @soi_retries = 5                                         # number of times to retry posting before giving up
  @host_no     = Facter.value('fqdn').split(/\./)[0].split(/0/)[1] # return server number, e.g nagios01.noc => 1
  @xmpp_pass   = "passwd"                                  # xmpp password
  @xmpp_host   = "openfire01.ops.lan:5222"   # xmpp host:port
  @xmpp_room   = "noise\@chat.int.mtnbusiness.net"         # xmpp chat room
  @xmpp_cmd    = " | sendxmpp -i -s 'nagios xmpp alert' -d -v -u nagbot-00#{@host_no} -p '#{@xmpp_pass}' -j #{@xmpp_host} -c #{@xmpp_room} -r nagbot-00#{@host_no}"  # send xmpp alert to chat

### Troubleshooting:
* Start the daemon:
  `/usr/local/etc/rc.d/nagbot_soi start`
* In a seperate console, watch the logfile: 
  `tail -f /var/spool/nagios/nagbot_soi.log`
* Check if it's running: 
  `[root@nagios03 ~/soi]# ps aux|grep nagbot_soi
root    90076  0.0  0.1 29712  9332   0  I     4:13PM   0:00.01 /usr/local/bin/ruby /usr/local/sbin/nagbot_soi.rb`
* Confirm if it's sending http data, run tcpdump on the  server: 
  `tcpdump -nXvvs 0 host 10.217.213.55 and port 80`
* You should see packets going out if you send a test alert directly to the fifo named pipe: 
  `echo "this is a CRIT message" > /var/spool/nagios/rw/ndbot.fifo`
* If are no packets, it's intermediary ACLs.
* If you do see packets outgoing but no responses inbound, confirm firewall rules.
* If you see responses going out, confirm that there are no issues with the SOI host with the Enterpise Monitoring team.
* For XMPP - 
  * If all seems well at the network level but you're not seeing any data in the actual jabber channel, check the bot and you are in the same channel.
  * Remember that if the bot joins the 'wrong' channel, that channel would be auto created into existence.

~~~ ruby 

#!/usr/bin/env ruby
# daemon to post nagios alerts (from a named pipe) to SOI environment 
# author: jayendren maduray <jayendren@gmail.com>

require 'rubygems'
require 'logger'
require 'uri'
require 'net/https'
require 'facter'

# global variables
# nagios
@nagios_fifo = "/var/spool/nagios/rw/ndbot.fifo"         # path to nagios named pipe
@log         = "/var/spool/nagios/nagbot_soi.log"        # path to logfile for this daemon
@log_size    = 1024000                                   # logfile size in kb
@logger      = Logger.new(@log, @log_size)               # logger instance
# soi
@soi_prot    = "http"                                    # http or https
@soi_host    = "soi.lan"                                 # ipv4/fqdn
@soi_api     = "Soi/default.aspx?"                       # api 
@soi_url     = "#{@soi_prot}://#{@soi_host}/#{@soi_api}" # soi url to post
@soi_timeout = 600                                       # number of seconds after to wait before giving up
@soi_retries = 5                                         # number of times to retry posting before giving up
# xmpp
@fqdn        = Facter.value('fqdn')
@xmpp_pass   = "passwd"                                   # xmpp password
@xmpp_host   = "openfire01.ops.lan\:5222"                 # xmpp host:port
@xmpp_room   = "noise\@chat.int.lan"          # xmpp chat room
@xmpp_cmd    = " | sendxmpp -i -s 'nagios xmpp alert' -d -v -u nagbot-00#{@host_no} -p '#{@xmpp_pass}' -j #{@xmpp_host} -c #{@xmpp_room} -r nagbot-00#{@host_no}"  # send xmpp alert to chat
# mail
@mail_rcpt   = "sysadmin@lan"
@mail_cmd    = " | mail -s 'nagbot_soi_error: #{@fqdn}' #{@mail_rcpt}"

# net/http post def
def nagbot_soi_post(url)

  begin
    uri                = URI.parse("#{url}")
    http               = Net::HTTP.new(uri.host, uri.port)
    http.read_timeout  = @soi_timeout

    unless url.to_s.match(/^http\:\/\/$/)
      http.use_ssl     = false
    else
      http.use_ssl     = true
    end

    http.verify_mode   = OpenSSL::SSL::VERIFY_NONE

    http.start {|http|
      request          = Net::HTTP::Get.new(uri.request_uri)
      @soi_response    = http.request(request)
    }

  rescue => e
    @logger.error e.message
    system("echo '#{e.message} - #{@soi_url}' #{@mail_cmd} > /dev/null 2>&1")
    e.backtrace.each { |line| 
      @logger.error line       
    }
    sleep 3
    retry if (@soi_retries -= 1) > 0
  end

end

# nagbot_soi daemon 
nagbot_soi = Proc.new do
  while true

    @spool = open(@nagios_fifo, "r+")     
    @alert = @spool.gets  

    begin
    rescue => e
      @logger.error e.message
      system("echo '#{e.message} - #{@nagios_fifo}' #{@mail_cmd} > /dev/null 2>&1")
      e.backtrace.each { |line| 
        @logger.error line         
      }
      exit 1
    end

    unless ! (@alert.to_s.match(/OK|UP|WARN|UNKNOWN|CRIT|DOWN/)) # add alert types here

      # set the sev state,
      # the following are the values of severities in SOI:
      # Normal   = 0
      # Minor    = 1
      # Major    = 2
      # Critical = 3
      # Down     = 4      

      if @alert.to_s.match(/OK|UP/) 
        @state = 0
      end
      if @alert.to_s.match(/WARN|UNKNOWN/)
        @state = 1
      end      
      if @alert.to_s.match(/CRIT/)
        @state = 3
      end
      if @alert.to_s.match(/DOWN/)
        @state = 4
      end    

      @data     = @alert.split("\s")                                                                    # convert entry in fifo to array
      @ci       = @data[0]                                                                              # extract hostname from array
      @sitmsg   = @data[1..-1].join(",").to_s.gsub(/\,/, " ")                                           # extract alert message from array

      # alarm id for clearing needs to match the critical alert
      begin
        a1              = Time.now.strftime("%d%m%Y")
        a2              = @alert.split("\s").to_s.bytes.to_a.to_s
        alert_timestamp = @alert.split(/check|server\-NRPE/)[1].split(/\s/)[0].bytes.to_a.to_s
        ci_timestamp    = (a2 + a1).to_i
        @alarm          = (ci_timestamp * alert_timestamp.to_i).to_s.chars.map(&:to_i)[0..4].to_s

        @soi_post = URI.encode("#{@soi_url}insid=#{@ci}&sitmsg=#{@sitmsg}&sev=#{@state}&alarm=#{@alarm}") # build the url to post
      rescue => e
        @logger.error e.message
        e.backtrace.each { |line| @logger.error line }
      end  

      @logger.info "alert           : #{@alert}"
      @logger.info "severity        : #{@state}"
      @logger.info "timestamp       : #{a1}"
      @logger.info "ci_timestamp    : #{ci_timestamp}"
      @logger.info "alert_timestamp : #{alert_timestamp}"
      @logger.info "alarm           : #{@alarm}"
      #@logger.info "posting         : #{@soi_post}"

      # soi post:
      #nagbot_soi_post(@soi_post)
      #@logger.info "reply           : #{@soi_response}"

      # xmpp post:
      @logger.info "xmpp            : echo '#{@alert}' #{@xmpp_cmd}"
      system("echo '#{@alert}' #{@xmpp_cmd} > /dev/null 2>&1")

    else
      @logger = Logger.new(@log, @log_size)
      @logger.info "ignoring: #{@alert}"

    end # unless

    sleep 1

    @spool.close

  end # while

end # Proc

fork { nagbot_soi.call }

~~~

RC Script
=========

    #!/bin/sh
    # bsd rc-compliant startup script for nagbot_soi ruby daemon

    # PROVIDE: nagbot_soi
    # REQUIRE: NETWORK
    # KEYWORD: shutdown

    #
    # Add the following lines to /etc/rc.conf to enable the puppet agent:
    #
    # nagbot_soi_enable="YES"


    . /etc/rc.subr
     
    name="nagbot_soi"
    rcvar=${name}_enable
     
    start_cmd="${name}_start"
    stop_cmd="${name}_stop"
     
    load_rc_config $name

    : ${nagbot_soi_enable="NO"}
    : ${nagbot_soi_rundir="/var/spool/nagios"}

    pidfile="${nagbot_soi_rundir}/${name}.pid"
    command="/usr/local/sbin/nagbot_soi.rb"
    command_interpreter="/usr/local/bin/ruby18"

    nagbot_soi_start() {
      if [ ! -f ${pidfile} ]; then
        echo -n "Starting services: ${name}"
        ${command_interpreter} ${command}
        sleep 5
        pid=`ps aux | grep ${command} | grep -v grep | awk '{print $2}'`
        echo ${pid} > ${pidfile}
        echo "."
       else
         echo "It appears ${name} is already running. NOT starting!"
      fi
    }
     
    nagbot_soi_stop() {
      if [ ! -f ${pidfile} ]; then
        echo "It appears ${name} is not running."
      else
        echo -n "Stopping services: ${name}"
        kill `cat ${pidfile}`
        rm ${pidfile}
        echo "."
      fi
    }

    run_rc_command "$1"
