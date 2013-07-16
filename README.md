event-rabbitmq-ex-rk-assignment
===============================

Assigning exchange and routing-key against rabbitmq-server from opensips1.9


1.3. RabbitMQ socket syntax
'rabbitmq:' [user[':'password] '@' host [':' port] '/' exchange

Meanings:

    'rabbitmq:' - informs the Event Interface that the events sent to this subscriber should be handled by the event_rabbitmq module.

    user - username used for RabbitMQ server authentication. The default value is 'guest'.

    password - password used for RabbitMQ server authentication. The default value is 'guest'.

    host - host name of the RabbitMQ server.

    port - port of the RabbitMQ server. The default value is '5672'.

    exchange - exchange name used by the AMQP protocol. It is NOT used to identify the queue.

1.5. Exported Parameters
1.5.1. routing-key_avp (str)

    This is for holding the name of the avp for assigning routing-key in the script. If the modparam is not used, the module will behave as before. The behavior is to use the default exchange of rabbitmq and the value of the "exchange" will be the routing-key of messages.


    Example 1.17. Set the “routing-key_avp” parameter

     ...
     modparam("dispatcher", "routing-key_avp", "$avp(routing-key)")
     ...

	

1.8.1. OpenSIPS config file



 startup_route {
 
    if (!subscribe_event("E_SIP_MESSAGE", "rabbitmq:127.0.0.1/sipmsg")) {
		xlog("L_ERR","cannot the RabbitMQ server to the E_SIP_MESSAGE event\n");
    }
  
 }


route{

  if (!mf_process_maxfwd_header("10")) {
		sl_send_reply("483","Too Many Hops");
		exit;
	}

	if (has_totag()) {
		if (loose_route()) {
			if (is_method("INVITE")) {
				record_route();
			}
			route(1);
		} else {
			if ( is_method("ACK") ) {
				if ( t_check_trans() ) {
					t_relay();
					exit;
				} else {
					exit;
				}
			}
			sl_send_reply("404","Not here");
		}
		exit;
	}

	if (is_method("CANCEL"))
	{
		if (t_check_trans())
			t_relay();
		exit;
	}

	t_check_trans();

	if (loose_route()) {
		xlog("L_ERR",
		"Attempt to route with preloaded Route's [$fu/$tu/$ru/$ci]");
		if (!is_method("ACK"))
			sl_send_reply("403","Preload Route denied");
		exit;
	}

	if (!is_method("REGISTER|MESSAGE"))
		record_route();

	if (!uri==myself)
	{
		append_hf("P-hint: outbound\r\n"); 
		route(1);
	}

	if (is_method("PUBLISH"))
	{
		sl_send_reply("503", "Service Unavailable");
		exit;
	}
	

	if (is_method("REGISTER"))
	{
		if (!save("location"))
			sl_reply_error();

		exit;
	}

	if ($rU==NULL) {
		sl_send_reply("484","Address Incomplete");
		exit;
	}

	if (is_method("MESSAGE")) {
		$avp(attrs) = "user";
		$avp(vals) = $rU;
		$avp(attrs) = "msg";
		$avp(vals) = $rb;
    
    # This message will be sent to exchange sipmsg with routing-key of the value of $tU
    # If the routing-key is not assigned, the message will be sent to the default exchange of the rabbitmq-server with the routing-key of the value of $tU. It means only the queue with the name of the value of $tU can get the message according to the amqp protocol. 
    $avp(routing-key) = $tU;
		if (!raise_event("E_SIP_MESSAGE", $avp(attrs), $avp(vals)))
			xlog("L_ERR", "cannot raise E_SIP_MESSAGE event\n");
	}

	if (!lookup("location","m")) {
		switch ($retcode) {
			case -1:
			case -3:
				t_newtran();
				t_reply("404", "Not Found");
				exit;
			case -2:
				sl_send_reply("405", "Method Not Allowed");
				exit;
		}
	}

	route(1);
}



