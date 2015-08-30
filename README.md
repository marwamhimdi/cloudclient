# cloudclient
import urllib2
import json
import datetime
from pprint import pprint
import sys

def fatal(message):
    print "ERROR ["+message+"]"
    exit(0)

def openurl(url):
    try:
        response = urllib2.urlopen(url).read()
        return response
    except Exception as e:
        fatal("Unable to open url "+url+" "+str(e))

def pack_sensor(sensor_id,temp_dict,humidity_dict,luminosity_dict,presence_dict):
    return {"_id" : sensor_id, "temperature" : temp_dict, "humidity" : humidity_dict, "luminosity" : luminosity_dict, "presence" : presence_dict}

def pack_controller(controller_id, sensors):
    d= {}
    d["id"] = controller_id
    d["sensors"] = sensors
    return {"controller" : d}

def getjson(url,method):
    url2 = url+method
    response = openurl(url2)
    try:
        data = json.loads(response)
    except Exception as e:
        fatal("Unable to get json from "+url2+" "+str(e))
    if data != None:
        return data
    else:
        fatal("Problem while rertrieving the json from "+url2)

def get_packed_data(url):
    temp = getjson(url,"get_temperature")
    humid = getjson(url,"get_humidity")
    lumin = getjson(url,"get_luminosity")
    pres = getjson(url,"get_presence")
    
    sensor_id = temp["sensor"]
    controller = temp["controller"]
    rmv_unused_fields(temp)
    rmv_unused_fields(humid)
    rmv_unused_fields(lumin)
    rmv_unused_fields(pres)
    
    return pack_sensor(sensor_id,temp,humid,lumin,pres)


def rmv_unused_fields(dict):
    del dict["controller"]
    del dict["type"]
    del dict["sensor"]

if __name__ == '__main__':
    if len(sys.argv) != 4:
        print "Usage : "+sys.argv[0]+" pi_ip_address server_ip_address username"
        exit(0)
    server_port = 5000
    
    server_base_url = "http://"+sys.argv[2]+":"+str(server_port)
    pi_base_url = "http://"+sys.argv[1]
    pi_list_url = pi_base_url+":32000/list"
    username = sys.argv[3]
    
    
    sensor_list = getjson(pi_list_url,"")
    s = []
    for e in sensor_list["sensor_ids"]:
        s.append(str(e))
    
    sensors = []
    for id in s:
        sensors.append(get_packed_data(pi_base_url+":32000/sensor/"+id+"/"))
    ctrl = pack_controller(sensor_list["controller"],sensors)
    print "Packed data to send :"
    pprint(ctrl)
    req = urllib2.Request(server_base_url+"/insertrecord/"+username)
    req.add_header('Content-Type', 'application/json')
    try:
        response = urllib2.urlopen(req, json.dumps(ctrl))
        print response.read()
    except Exception as e:
        fatal("Unable to communicate with the server "+str(e))


