##  API for converting speech to text file 
Template repository for all the projects.
import xml.etree.ElementTree as XmlElementTree	
import httplib2							
import uuid							
from config import ***
 
***_HOST = '***'
***_PATH = '/***_xml'
CHUNK_SIZE = 1024 ** 2
 
## функция дла принятия на вход файла размером до1 мб и реализация передачи по частям
def speech_to_text(filename=None, bytes=None, request_id=uuid.uuid4().hex, topic='notes', lang='ru-RU',
               	key=***_API_KEY):
  
## Если передан файл
	if filename:
    	with open(filename, 'br') as file:
        	bytes = file.read()
	if not bytes:
    	raise Exception('Neither file name nor bytes provided.')
 
  ## Конвертирование в нужный формат
	bytes = convert_to_pcm16b16000r(in_bytes=bytes)
 
## Формирование тела запроса к API
	url = ***_PATH + '?uuid=%s&key=%s&topic=%s&lang=%s' % (
    	request_id,
    	key,
    	topic,
    	lang
	)
 
## Считывание блока байтов
	chunks = read_chunks(CHUNK_SIZE, bytes)
 
## Установка соединения и формирование запроса
	connection = httplib2.HTTPConnectionWithTimeout(***_HOST)
 
	connection.connect()
	connection.putrequest('POST', url)
	connection.putheader('Transfer-Encoding', 'chunked')
	connection.putheader('Content-Type', 'audio/x-pcm;bit=16;rate=16000')
	connection.endheaders()
 
  ## Отправка байтов блоками
	for chunk in chunks:
    	connection.send(('%s\r\n' % hex(len(chunk))[2:]).encode())
    	connection.send(chunk)
    	connection.send('\r\n'.encode())
 
	connection.send('0\r\n\r\n'.encode())
	response = connection.getresponse()
 
## Обработка ответа сервера
	if response.code == 200:
    	response_text = response.read()
    	xml = XmlElementTree.fromstring(response_text)
 
    	if int(xml.attrib['success']) == 1:
        	max_confidence = - float("inf")
        	text = ''
 
        	for child in xml:
            	if float(child.attrib['confidence']) > max_confidence:
                	text = child.text
                	max_confidence = float(child.attrib['confidence'])
 
	        if max_confidence != - float("inf"):
            	return text
        	else:
## выброска исключений если текс не найден или неизвестная ошибка
            	raise SpeechException('No text found.\n\nResponse:\n%s' % (response_text))
    	else:
        	raise SpeechException('No text found.\n\nResponse:\n%s' % (response_text))
	else:
    	raise SpeechException('Unknown error.\nCode: %s\n\n%s' % (response.code, response.read()))
 
## исключение
сlass SpeechException(Exception):
	pass
