## We have a script that we use to generate QR codes. It needs two inputs the message/URL and the file name for the QR code. Please look at how to use the argparse module to set it in a way that we can run it with arguments and provide those inputs. -h or --help should show how to use the script, -u or --url should be used to provide a URL and -i or --im should be used to provide the file image name.

### the script is below:

```python
# install pyqrcode and pypng
import pyqrcode
import png
from pyqrcode import QRCode

# Text which is to be converted to QR code
text_to_convert=input("Enter text to convert: ")
# Name of QR code png file
image_name=input("Enter image name to save: ")
# Adding extension as .pnf
file_name=image_name+".png"
# Creating QR code
url=pyqrcode.create(text_to_convert)
# Saving QR code as  a png file
url.show()
url.png(file_name, scale =6)


### solution:

```python

# install pyqrcode and pypng
import pyqrcode
import png
from pyqrcode import QRCode
import argparse

# Text which is to be converted to QR code
text_to_convert=input("Enter text to convert: ")
# Name of QR code png file
image_name=input("Enter image name to save: ")
# Adding extension as .pnf
file_name=image_name+".png"
# Creating QR code
url=pyqrcode.create(text_to_convert)
# Saving QR code as  a png file
#url.show()
url.png(file_name, scale =6)

parser = argparse.ArgumentParser()
parser.add_argument('-u', help='the url of the qrcode', action="store_true")
parser.add_argument('--url', help='the url of the qrcode', action="store_true")
parser.add_argument('-i', help='The name of the image', action="store_true")
parser.add_argument('--im', help='The name of the image', action="store_true")
args = parser.parse_args()

if args.im or args.i:
    print(f"The name of the file is {file_name}")

if args.url or args.u:
    url.show()
