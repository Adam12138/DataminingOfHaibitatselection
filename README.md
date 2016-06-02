# DataminingOfHaibitatselection
This project is about mining habitat selection of birds based on its flying location data sampling one by 2 hours.


import MySQLdb
from pylab import *
from osgeo import gdal
import sys
import Image
import random
import scipy.stats as st

def connectDB():
    conn = MySQLdb.connect(
        host="127.0.0.1",
        port=3306,
        user="root",
        passwd="123456",
        db="birdgps"
    )
    cur = conn.cursor()
    cur.execute("select latitude from viewlg where ptt = 74822")
    x = cur.fetchall()
    cur.execute("select longitude from viewlg where ptt = 74822")
    y = cur.fetchall()
    cur.close()
    conn.commit()
    conn.close()
    return (x, y)


def transfer(x, y):
    latitude = []
    longitude = []
    for a in x:
        latitude.append(a[0])
    print latitude
    for b in y:
        longitude.append(b[0])
    print longitude
    return (latitude, longitude)


def geoTransform(srcImage, latitude, longitude):
    geoTrans = srcImage.GetGeoTransform()
    ulX = geoTrans[0]
    ulY = geoTrans[3]
    xDist = geoTrans[1]
    yDist = geoTrans[5]
    band = srcImage.GetRasterBand(1)
    s = float(ulX)
    t = float(ulY)
    print s
    print t
    p = float(xDist)
    q = float(yDist)
    print p
    print q
    locationOfLatitude = []
    locationOfLongitude = []
    pixlati = []
    pixlongi = []
    for m in latitude:
        num1 = float((m - ulY) / yDist - 2000)
        nums1 = float((m - ulY) / yDist)
        locationOfLatitude.append(num1)
        pixlati.append(nums1)
    m = len(locationOfLatitude)
    print m
    for n in longitude:
        num2 = float((n - ulX) / xDist - 2500)
        nums2 = float((n - ulX) / xDist)
        locationOfLongitude.append(num2)
        pixlongi.append(nums2)
    # print locationOfLongitude
    return (m, locationOfLatitude, locationOfLongitude, pixlati, pixlongi, band)


def getValue(m, latitude, longitude, band):
    i = 0
    values = []
    while i < m:
        p = int(latitude[i])
        q = int(longitude[i])
        data = band.ReadAsArray(q, p, 1, 1)
        value = data[0, 0]
        values.append(value)
        i += 1
    print values
    return values


def random1(m):
    rlatitude = []
    rlongitude = []
    rlatitude1 = []
    rlongitude1 = []
    i = 0
    while (i < m):
        s = random.uniform(0, 2000)
        rlatitude.append(s)
        t = random.uniform(0, 4000)
        rlongitude.append(t)
        i += 1
    k = 0
    while k < m:
        rlatitude1.append(rlatitude[k] + 2000.0)
        k += 1
    j = 0
    while j < m:
        rlongitude1.append(rlongitude[j] + 2500.0)
        j += 1
    return (rlatitude, rlongitude, rlatitude1, rlongitude1)


def show1(p, q):
    plot(p, q, 'r*')
    # plot(p[:], q[:])
    title('Plotting:"water"')
    show()
    return 1


def categoryGrade(values):
    grade02 = []
    grade24 = []
    grade46 = []
    grade6u = []
    x = []
    for i in values:
        if i <= 2000.0:
            grade02.append(i)
        elif 2000.0 < i <= 4000.0:
            grade24.append(i)
        elif 4000.0 < i <= 6000.0:
            grade46.append(i)
        else:
            grade6u.append(i)
    x.append(float(len(grade02)))
    x.append(float(len(grade24)))
    x.append(float(len(grade46)))
    x.append(float(len(grade6u)))
    return x


def o_pi_calcu(m, x):
    u = []
    i = 0
    while i < 4:
        u.append(x[i] / m)
        i += 1
    return u


def chisquare1(m, n, wi, countNum, rcountNum, o, pi):
    Z = st.norm.ppf(1 - 0.05 / (2 * n))
    seWiList = []  # s.e value of wi
    interval = []
    chi2 = 0
    selection = ['+', '-', '0']

    print '----------------------------------------------------'

    for i in range(n):

        if (countNum[i] == 0):
            countNum[i] = 1
            o[i] = float(1.0 / m)
        if (rcountNum[i] == 0):
            rcountNum[i] = 1
            pi[i] = float(1.0 / m)

        wiTemp = float(o[i] / pi[i])
        wi.append(wiTemp)
        seTemp = float(wiTemp * float(math.sqrt(1.0 / countNum[i] - 1.0 / m + 1.0 / rcountNum[i] - 1.0 / m)))

        seWiList.append(seTemp)

        intervalTemp = [wiTemp - Z * seTemp, wiTemp + Z * seTemp]
        interval.append(intervalTemp)
        chi2T = float(countNum[i]) * math.log(float(countNum[i]) / (float(countNum[i] + rcountNum[i]) * m / (m + m))) \
                + float(rcountNum[i]) * math.log(
            float(rcountNum[i]) / (float(countNum[i] + rcountNum[i]) * m / (m + m)))
        chi2 += chi2T * 2

        # selection jugement
        if (intervalTemp[0] > 1):
            a = selection[0]
        else:
            if (intervalTemp[1] < 1):
                a = selection[1]
            else:
                a = selection[2]

        print 'Category%2d:   Wi:%-9.3f s.e.(Wi):%9.3f   interval:(%9.3f   -- %9.3f)   selection:%4s' % (
        i + 1, wiTemp, seTemp, intervalTemp[0], intervalTemp[1], a)


def main(raster_path):
    srcImage = gdal.Open(raster_path)
    if srcImage is None:
        print 'Could not find the RasterData'
        sys.exit(1)
    land = array(Image.open(raster_path))
    zland = land[2000:4000, 2500:6500]
    imshow(zland)
    x, y = connectDB()
    latitude, longitude = transfer(x, y)
    m, locationOfLatitude, locationOfLongitude, pixlati, pixlongi, band = geoTransform(srcImage, latitude, longitude)
    values1 = getValue(m, pixlati, pixlongi, band)
    countNum = categoryGrade(values1)
    o = o_pi_calcu(m, countNum)
    rlatitude, rlongitude, rlatitude1, rlongitude1 = random1(m)
    valuesr = getValue(m, rlatitude1, rlongitude1, band)
    rcountNum = categoryGrade(valuesr)
    pi = o_pi_calcu(m, rcountNum)
    print o[0], o[1], o[2], o[3], pi[0], pi[1], pi[2], pi[3]
    wi = []
    i = 0
    while i < 4:
        wi.append(o[i] / pi[i])
        i += 1
    print wi[0], wi[1], wi[2], wi[3]
    print '**********************************************'
    n = 4
    chisquare1(m, n, wi, countNum, rcountNum, o, pi)
    print '**********************************************'
    show1(locationOfLongitude, locationOfLatitude)
    imshow(zland)
    show1(rlongitude, rlatitude)


if __name__ == '__main__':
    if len(sys.argv) < 1:
        print "[ ERROR ] you must args. the full raster path"
        sys.exit(1)
    raster_path = ('C:/Users/buffer/Desktop/water.tif')
    main(raster_path)
