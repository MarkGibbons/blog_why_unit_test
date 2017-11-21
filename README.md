# Why unit test

Take this code from the rhn_satellite_api gem used to manage channel subscriptions and patch RHEL servers.

The output from system.getID  looks like [{id: string, name: string, last_checkin: XMLRPC::last_checkin}, repeat hash ]. That result is an array of hashes.

````
def get_systemid_by_hostname (session, key, hostname)
    sorted_server_list = session.call('system.getId', key, hostname).sort_by 
      { |id, name, last_checkin| last_checkin }.reverse
    current_system = sorted_server_list.last
end
````

In theory, this code looks up all the hosts with a given name in rhn_satellite and returns an array.  Without some decent unit tests you might think this code makes sense.  However, sort_by on an array just fills in the first value, id with the hash entry.  The rest of the key values are set to nil. The sort evaluates nil for each entry and returns a random host.  Not necessarily the host you were trying to patch.  The code will work most of the time because most host are defined in the rhn satellite exactly once.

Let’s fix it, it can’t be that hard. Let’s sort on one of the hash values.

````
def get_systemid_by_hostname (session, key, hostname)
    sorted_server_list = session.call('system.getId', key, hostname).sort_by 
      { |server| server[‘last_checkin’] }.reverse
    current_system = sorted_server_list.last
end
````

One interesting thing is the XMLRPC::last_checkin structure does not implement <> so the sort gets a missing method return and fails.  Better to fail than give incorrect results in my opinion.  But you don’t see this error without unit tests.  The hash entries themselves are also not directly sortable.  Hash doesn’t implement a direct comparison of hashes.

Let’s fix it some more:

````
def get_systemid_by_hostname (session, key, hostname)
    sorted_server_list = session.call('system.getId', key, hostname).sort_by 
      { |server| real_date(server[‘last_checkin’]) }.reverse
    current_system = sorted_server_list.last
end
````

Write a method to extract the date from server[‘last_checkin’].  Guess what that reverse sorts in descending order and last picks the last/oldest server.  We almost always will want the newest in this situation. Instead, we pick exactly the oldest server every time. Unit tests again.  

Sure, writing tests looks like more work, but simple logic can be hard and it’s easy to fool yourself into believing you did something right because it works most of the time.  Debugging this code has sucked up way more time than writing some unit tests up front during development. The session.call('system.getId') code will need a mock that returns test data and testing edge cases, empty return, single entry, multiple entries, duplicate entries, entries missing the key values may be more than you signed up for.  But a happy case test with multiple values returned in some random order would have found most of the problems.

