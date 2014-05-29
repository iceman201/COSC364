''' COSC364 Assignment 1
Written by Christopher Flores and Liguo Jiao
Due 16-May-2014 '''

import socket
import select
import sys
import time
import random

# Global Variables: Information from the Config
own_id = -1
output_ports = {}
input_ports = []

def print_table(table):
    ''' Prints the routing table in a readable format
        @param table: dictionary representing the routing table '''
    routerids = sorted(table.keys())
    print("Own Router ID: {0}".format(own_id))
    print(" ID ||First|Cost|Grbg Flag|Drop t.out|Grbg t.out|")
    print("----++-----+----+---------+----------+----------+--")
    for i in range(0, len(table)):
#         print( str(routerids[i]) + " : " + str(table[routerids[i]]) )
        info = table[routerids[i]]
        print(" {:>2} ||".format(routerids[i]), end='')
        print(" {:>3} | {:>2} | {:<7} | {:<8} | {:<8} |"\
              .format(info[0], info[1], info[2], str(info[3][0])[:8], str(info[3][1])[:11]))

def first_hop(frouter, table):
    ''' Returns a list of routers whose routes in the routing table use
        'frouter' as the first hop'''
    routers = []
    for router in sorted(table.keys()):
        if table[router][0] == frouter:
            routers.append(router)
    return routers

def routing_table(filenm):
    ''' Reads the config from the config file and builds the routing table
        from it. 
        @param filename: name of the config file to read '''
    global own_id

#     print("----- Reading Configuration File")
    total = []
    table = {}
    c1=open(filenm, 'r')
    for i in c1.readlines():
        i=i.split(', ')
        total.append(i)
#     print(total[0]) 
    own_id = int(total[0][1])
#     print("Router-id: " + str(own_id))
        
    for i in range(1, len(total[1])):
        input_ports.append(int(total[1][i]))
#     print ("Input Ports: " + str(input_ports))

    for i in range(1,len(total[2])):
        temp = total[2][i]
        temp = temp.split('-')
        router_id = int(temp[-1])
        portno = int(temp[0])
        output_ports[portno] = router_id
        first_router = router_id
        metric = int(temp[-2])
        flag = False
        timers = [0, 0]
                
        # Layout of this table is [ID: [FIRST-ROUTER,COST,FLAG,TIMER]]
        table[router_id] = [first_router, metric, flag, timers] 
#     print("Output Ports: " + str(output_ports))
#     print_table(table)
#     print("----- Finished reading configuration file.\n")
    return table

def router_hashkey(t):
    ''' Returns a list of keys from the routing table
        @param t: dictionary representing the routing table '''
    hashkey=[]
    table = t
    if table != None:
        for y in table.keys():
            hashkey.append(y)
        return hashkey
    else:
        return hashkey

def listenlist():
    l_list=[]
    for i in range(0, len(input_ports)):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.bind(('', int(input_ports[i]) ))        
        l_list.append(sock)
    return l_list

def id_in_list(rec_id, table):
    ''' Checks whether or not the given router_id exists in the routing
        table
        @param rec_id: ID of the router we are checking
        @param table: dictionary representing the routing table '''

    key = router_hashkey(table)
    if rec_id in key:
        return True
    else:
        return False

def receiver(rt_table, timeout):
    ''' Checks the sockets to see if any messages have been received.
        @param rt_table: dictionary representing the routing table
    '''
#     print("--- Checking for packets")
    socket_list = listenlist() 
    table_key = []
    table = rt_table
    a,b,c = select.select(socket_list,[],[], timeout)
    if a != []:
        s = a[0]
        s.settimeout(10)
        data,addr = s.recvfrom(1024)
        print("----- Received message: " + str(data))
        data = str(data)
        data = data.replace("b","").replace("'","")
        data = data.split(',')
        src = int(data[2])

        # reset timers for direct neighbour
        table[src][-1][0] = 0 
        table[src][-1][1] = 0
        table[src][2] = False

        start = 3
        while start < len(data):
            rec_id = int(data[start]) 
            router_id = int(data[start])
            metric = min(int(data[start+1]) + table[src][1], 16) #TODO: KeyError:1
            
            if metric not in range(0,17):
                print("Packet does not conform. Metric not in range 0-16")
                break

            if router_id not in output_ports.values() and router_id != own_id:
                if not id_in_list(rec_id, table):
                    table[router_id] = [src, metric, False, [0,0]]

                if (metric < table[router_id][1]):
                    # better route has been found
                    table[router_id][1] = metric 
                    table[router_id][0] = src

                if (src == table[router_id][0]):
                    # Submitting to the information the router gives us if they
                    # are the first hop to the destination.
                    table[router_id][1] = metric 
                    table[router_id][0] = src
                    table[router_id][-1][0] = 0 # Reset Timer
                    table[router_id][-1][1] = 0
                    table[router_id][2] = False

            start += 2
    return table

def create_message(t, port):
    ''' Creates message with information about
        @param t: dictionary representing the routing table
        @param port: port number that it is being sent to so that we can 
                     exclude entries.
    '''

    table = t # {}
    command = "2" # Command No.: 2
    version = "2" # Version No.: 2
    source = own_id
    key = router_hashkey(table)
    head = command + ',' + version + ','+ str(source)
    result = head
   
    for i in key:
        value = table[i]
        result += ',' + str(i) + ','
        metric = str(value[1]) if i not in table.values() else 16
        result += metric
#     print("Created message to send: " + str(result))
    return result

def send_message(table):
    ''' Sends an update message to all direct neighbours
        @param table: dictionary representing the routing table '''
#     print("----- Sending update message to all direct neighbours")
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    for port in output_ports:  
        msg = create_message(table, port)
        msg = msg.encode('utf-8')
    
        sock.sendto(msg, ("0.0.0.0", port))
#         if (sock.sendto(msg, ("0.0.0.0", port))):
#             print("Sent packet to port: " + str(port))
#     print("----- Finished sending update message.\n")

def update_timers(table, time):
    ''' Adds time onto all routing table entry timers.
        @param table: dictionary representing the routing table
        @param time: time to add to the timers. '''
    route_invalid_timeout = 30
    garbage_collection_timeout = 24
#     print("UPDATING TIMERS")
    for key in sorted(table.keys()):
#         print("incrementing timer")
        if table[key][2]:
            table[key][-1][1] += time
            if table[key][-1][1] > garbage_collection_timeout:
                del table[key]
                # remove entry from dictionary
        else:
            table[key][-1][0] += time
            if table[key][-1][0] > route_invalid_timeout:
#                print("set metric to 16")
                 table[key][1] = 16 # Set route to timed out router to infinity
                 table[key][2] = True
                 # Do the same for routes whose first hop is 'key'
                 for router in first_hop(key, table):
                    table[router][1] = 16
    print("\n")

def run():
    counter = 0
    rt_table = routing_table(sys.argv[1])

    while 1:
        print("loop " + str(counter))
        maxtime = 2 + random.randint(-2,2)
        timeout = maxtime
        track = time.time()
        elapsed = track - time.time()
        timer_incr = 0

        while elapsed < maxtime:
            rt_table = receiver(rt_table, timeout) 
            timer_incr = time.time() - track
            track = time.time()
            update_timers(rt_table, timer_incr)
            elapsed += timer_incr
            timeout = max(maxtime - elapsed, 0)

        print_table(rt_table)
        print("\n")
        send_message(rt_table)
        counter += 1

if __name__ == "__main__":
    n_args = (len(sys.argv))
    if n_args == 2: run()
    else: print("Daemon takes exactly one configuration file.") 
