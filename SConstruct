from rsf.proj import *


def section(title,label1='Time',unit1='s',min1=5.8,max1=8.0,extra=" "):
    return '''
    window min1=%g max1=%g |
    grey title="%s"
    label1="%s" unit1="%s" label2=Distance unit2=m %s
    ''' % (min1,max1,title,label1,unit1,extra)

# Download data

Fetch('Nshots.su','nankai')

# Convert from SU to RSF

Flow('shots tshots','Nshots.su',
     	'''
	suread suxdr=y tfile=${TARGETS[1]} 
	''')

Result('spectra','shots',
       'spectra all=y |  scale axis=1 | graph title="Average Spectra"')

Fetch('Nstack.su','nankai')

Flow('stackd tstackd','Nstack.su','suread suxdr=y tfile=${TARGETS[1]}')
Flow('stack-win','stackd','window min1=5')

Plot('stackd','window j1=2 j2=2 | grey title=Stack')
Plot('stack-win','window  j2=2 | grey title="Stack (Window)" ')

Result('stackd','stackd stack-win','SideBySideAniso')

####### DC Removal

Flow('mean','shots','stack axis=1 | spray axis=1 n=5500 o=0.0 d=0.002')

Flow('shotsdc','shots mean','add scale=1,-1 ${SOURCES[1]}')


####### Bandpass Filtering

Flow('shotsf','shotsdc','bandpass flo=10')

####### Windowing Data

Flow('shotsw','shotsf','window min1=5.5')

####### Mask zero traces

Flow('mask0','shotsw','mul $SOURCE | stack axis=1 | mask min=1e-20')

Flow('shots0','shotsw mask0','headerwindow mask=${SOURCES[1]}')

# update a database
Flow('tshots0','tshots mask0','headerwindow mask=${SOURCES[1]}')

####### Surface consistent

# Average trace amplitude
Flow('arms','shots0',
     'mul $SOURCE | stack axis=1 | math output="log(input)"')

# shot/offset indeces: fldr and tracf

Flow('indexshot','tshots0','window n1=1 f1=2')


Flow('offsets4index','tshots0',' headermath output=offset | dd type=float | window')

Flow('offsetindex','offsets4index','math output="abs(input) - 170" | dd type=int')

# receiver/midpoint

Flow('midpoint','tshots0','window n1=1 f1=5')

Flow('cmps4index','tshots0',' headermath output=cdp | dd type=float | math output="input*16.667" | window')

Flow('recv','cmps4index offsets4index','add scale=1,0.5 ${SOURCES[1]} | math output="input - 13799" | dd type=int')

Flow('index','indexshot offsetindex',
        '''
        cat axis=2 ${SOURCES[1]}
        ''')

Flow('extindex','index midpoint',
        '''
        cat axis=2 ${SOURCES[1]}
        ''')

Flow('extindrecv','extindex recv',
        '''
        cat axis=2 ${SOURCES[1]}
        ''')

def plot(title):
    return '''
    spray axis=1 n=1 | 
    intbin head=${SOURCES[1]} yk=fldr xk=tracf | window | 
    grey title="%s" label2="Shot Number" unit2= 
    label1="Offset Number" unit1= scalebar=y
    ''' % (title)

def plotb(title,bias=-5):
    return '''
    spray axis=1 n=1 | 
    intbin head=${SOURCES[1]} yk=fldr xk=tracf | window | 
    grey title="%s" label2="Shot Number" unit2= 
    label1="Offset Number" unit1= scalebar=y clip=3 bias=%g
    ''' % (title,bias)

# Display in shot/offset coordinates
Flow('varms','arms tshots0','spray axis=1 n=1 | intbin head=${SOURCES[1]} yk=fldr xk=tracf | window')
Result('varms','arms tshots0',plotb('Log-Amplitude'))

prog = Program('surface-consistent.c')
sc = str(prog[0])

# recv index

# get model dimensions
Flow('recvmodel',['arms','extindrecv',sc],
     './${SOURCES[2]} index=${SOURCES[1]} verb=y')

# find a term
Flow('recvsc',['arms','extindrecv',sc,'recvmodel'],
     '''
     conjgrad ./${SOURCES[2]} index=${SOURCES[1]} 
     mod=${SOURCES[3]} niter=150
     ''')

# project to a data space
Flow('recvscarms',['recvsc','extindrecv',sc],
     './${SOURCES[2]} index=${SOURCES[1]} adj=n')

Result('recvvscarms','recvscarms tshots0',
       plotb('Source, Offset, CDP, Recv S-C Log(A)'))

# compute difference
Flow('recvadiff','arms recvscarms','add scale=1,-1 ${SOURCES[1]}')

Result('recvadiff','recvadiff tshots0',plot('s,h,cdp,r difference'))

size=dict(sht=326,off=2266,rcv=7784,cmp=401)
f1 = 0
for case in ('sht','off','rcv','cmp'):
    n1=size[case]
    
    Result(case,'recvsc',
           '''
           window n1=%d f1=%d | put o1=1 d1=1 | 
           graph title="%s Term" 
           label1="%s Number" unit1= label2=Amplitude unit2=
           ''' % (n1,f1,case.capitalize(),case.capitalize()))
    f1 += n1

### apply to traces to all times - no windowing is considered

Flow('ampl','recvscarms',
     'math output="exp(-input/2)" | spray axis=1 n=5500 d=0.002 o=0')

Flow('shotsf0','shotsf mask0','headerwindow mask=${SOURCES[1]}')

Flow('shots-preproc','shotsf0 ampl','mul ${SOURCES[1]}')

Plot('shots-preproc','shots-preproc','window n2=100 | grey min1=6.0 max1=8.0 title="Shots Preproc"')

Plot('shots-raw','shots0','window n2=100 | grey min1=6.0 max1=8.0 title="Shots Raw"')


# Resample to 4 ms

Flow('subsamples','shots-preproc',
     '''
     bandpass fhi=125 | window j1=2
     ''')

Result('spectrasub','subsamples',
       'spectra all=y | graph title="Subsampled Spectra"')

Result('spectra-check','shots-preproc',
       'spectra all=y | graph max1=160 title="Spectra Check"')

# Extract shots
Flow('shots2','subsamples tshots0',
        '''
        intbin xk=tracf yk=fldr head=${SOURCES[1]} 
        ''')
Result('shots2',
       '''
       window min1=5.5 max1=9 | byte gainpanel=all | transp plane=23 memsize=5000 |
       grey3 frame1=300 frame2=100 frame3=50 
       point1=0.8 point2=0.8 clip=8.66
       title="Shots" label3="Shots" 
       ''')


# Create a mask to remove misfired shots

Flow('smask','shots2','mul $SOURCE | stack axis=1 | mask min=1e-20')

Result('smask',
       '''
       dd type=float |
       stack axis=1 norm=n |
       graph symbol=x title="Nankai Trough"
       label2="Traces per Shot Gather"
       label1="Shot Number" unit1=
       ''')

Flow('offsets','tshots0',
     '''
     window n1=1 f1=11 squeeze=n | dd type=float |
     intbin head=$SOURCE xk=tracf yk=fldr
     ''')


# Select one shot (fldr)

shot=1707

Flow('mask','smask','window n2=1 min2=%g' % shot)

Flow('shot','shots2 mask',
     '''
     window n3=1 min3=%g squeeze=n |
     headerwindow mask=${SOURCES[1]}
     ''' % shot)

Flow('offset','offsets mask',
     '''
     window n3=1 min3=%g squeeze=n |
     headerwindow mask=${SOURCES[1]}
     ''' % shot)

Result('shot','shot offset',
       '''
       window min1=5.8 max1=8.5 |
       wiggle xpos=${SOURCES[1]} yreverse=y transp=y poly=y
       title="Shot 1707" label2=Offset
       ''')

Plot('shot',' window min1=5.8 max1=8.5 | grey title="Selected Shot" clip=2')

# Extract CMPs and apply t^2 gain

Flow('cmps maskcmp','subsamples tshots0',
     '''
      intbin head=${SOURCES[1]} mask=${TARGETS[1]} 
      xk=tracf yk=cdp 	      |
      pow pow1=2 	
      ''')

Result('cmps',
        '''
        window min1=5.8 max1=8.5 |
	byte gainpanel=all | transp plane=23 memsize=5000 |
        grey3 frame1=400 frame2=50 frame3=200 
	title="CMPs" label2="CMP Number"
        flat=n point1=0.8 point2=0.8
	''')

Flow('cmask','cmps','mul $SOURCE | stack axis=1 | mask min=1e-20')

Result('cmask',
       '''
       dd type=float |
       stack axis=1 norm=n |
       graph symbol=x title="Nankai Trough"
       label2="Traces per CMP Gather"
       label1="CMP Gather Number" unit1=
       ''')

Flow('offs','tshots0',
     '''
     window n1=1 f1=11 squeeze=n | dd type=float |
     intbin xk=tracf yk=cdp head=$SOURCE
     ''')

# Examine one CMP gather

Flow('mask1','cmask','window n2=1 min2=1280')

Flow('cmp1','cmps mask1',
     '''
     window n3=1 min3=1280 | headerwindow mask=${SOURCES[1]}
     ''')

Flow('off','offs mask1',
     '''
     window n3=1 min3=1280 squeeze=n | headerwindow mask=${SOURCES[1]}
     ''')

Result('cmp1','cmp1 off',
     '''
     window min1=5.8 max1=8.5 |
     wiggle xpos=${SOURCES[1]} title="CMP 1280"
     yreverse=y transp=y poly=y label2=Offset unit2=m
     wherexlabel=t wheretitle=b
     ''')

Plot('cmp1','cmp1 off',
     '''
     window min1=5.8 max1=8.5 |
     wiggle xpos=${SOURCES[1]} title="CMP 1280"
     yreverse=y transp=y poly=y label2=Offset unit2=m
     wherexlabel=t wheretitle=b
     ''')

# Velocity analysis and NMO

Flow('vscan','cmp1 off mask1',
     '''
     vscan half=n offset=${SOURCES[1]} 
     v0=1400 nv=101 dv=10 semblance=y 
     ''')

Plot('vscan',
     '''
     window min1=5.8 max1=8.5 | 
     grey color=j allpos=y title="Velocity Scan" unit2=m/s
     ''')

Flow('pick','vscan',
     '''
     mutter inner=y half=n t0=5 x0=1400 v0=75 | 
     pick v0=1500 rect1=25
     ''')

Plot('pick',
     '''
     window min1=5.8 max1=8.5 |
     graph transp=y yreverse=y plotcol=7 plotfat=3
     pad=n min2=1400 max2=2400 wanttitle=n wantaxis=n
     ''')

Plot('vscanp','vscan pick','Overlay')

Flow('nmo','cmp1 off mask1 pick',
     '''
     nmo half=n offset=${SOURCES[1]} 
     velocity=${SOURCES[3]}
     ''')

Result('nmo','window min1=5.8 max1=8.5 | grey title="Normal Moveout"')

Plot('cmpg','cmp1',
     '''
     window min1=5.8 max1=8.5 | 
     grey title="CMP 1280" labelsz=12 titlesz=18
     ''')

Plot('nmog','nmo',
     '''
     window min1=5.8 max1=8.5 | 
     grey title="Normal Moveout" labelsz=12 titlesz=18
     ''')

Result('nmo1','cmpg nmog','SideBySideAniso')

Result('vscan','cmp1 vscanp','SideBySideAniso')

# Apply to all CMPs

Flow('cmask2','cmask','transp plane=23 | transp plane=21')

Flow('vscans','cmps offs cmask2',
     '''
     vscan half=n offset=${SOURCES[1]} mask=${SOURCES[2]}
     v0=1400 nv=101 dv=10 semblance=y nb=5
     ''',split=[3,'omp'])

Flow('picks','vscans',
     '''
     mutter inner=y half=n t0=5 x0=1400 v0=75 |
     pick v0=1500 rect1=25 rect2=10
     ''')

Result('picks',
       '''
       window | grey color=j mean=y scalebar=y title="NMO Velocity"
       label2=CMP barreverse=y barlabel=Velocity barunit=m/s
       ''')

Flow('nmos','cmps offs cmask picks',
     '''
     nmo half=n offset=${SOURCES[1]} mask=${SOURCES[2]}
     velocity=${SOURCES[3]}
     ''')

Result('nmos',
        '''
        window min1=5.5 max1=8.5 |
	    byte gainpanel=all | transp plane=23 memsize=5000 |
        grey3 frame1=400 frame2=50 frame3=200 
	    title="NMO" label2="CMP Number"
        flat=n point1=0.8 point2=0.8
	    ''')

Flow('stack','nmos','stack')

Result('stack',
       '''
       window min1=5.5 max1=8.5 | 
       grey title="NMO Stack" label2="CMP Number" 
       ''')

Plot('stack',
     '''
     window min1=5.5 max1=8.5 | 
     grey title="NMO Stack" label2="CMP Number" 
     ''')

# Try DMO

nv=60

Flow('stacks','cmps offs maskcmp',
     '''
     stacks half=n v0=1400 nv=%g dv=20 
     offset=${SOURCES[1]} mask=${SOURCES[2]}
     '''%nv, split=[3,'omp'])
     
Flow('stackst','stacks','costaper nw3=20')

Result('stacks','stackst',
       '''
       byte gainpanel=all | transp plane=23 memsize=5000 |
       grey3 frame1=400 frame2=50 frame3=200 point1=0.8 point2=0.8
       title="Constant-Velocity Stacks" label3=Velocity unit3=m/s
       label2="CMP Number"
       ''')

# Apply double Fourier transform (cosine transform)

Flow('cosft3','stackst',
     '''
     put d3=16.667 o3=0 label2=Distance unit2=m | 
     cosft sign1=1 sign3=1
     ''')

# Transpose f-v-k to v-f-k

Flow('transp','cosft3','transp')

# Fowler DMO: mapping velocities

Flow('map','transp',
     '''
     math output="x1/sqrt(1+0.25*x3*x3*x1*x1/(x2*x2))" | 
     cut n2=1
     ''')
        
Flow('fowler','transp map','iwarp warp=${SOURCES[1]} | transp')


# Inverse Fourier transform

Flow('dmo','fowler','cosft sign1=-1 sign3=-1')
 
Result('dmo',
       '''
       put d3=1 o3=900 unit2='' label2=Trace | 
       byte gainpanel=all | transp plane=23 memsize=5000 |
       grey3 frame1=400 frame2=50 frame3=200 point1=0.8 point2=0.8
       title="Constant-Velocity DMO Stacks" 
       label3=Velocity unit3=m/s 
       ''')
       
# Compute envelope for picking

Flow('envelope','dmo','envelope | scale axis=2')

# Mute and Pick velocity

Flow('vpick','envelope',
    '''
	mutter v0=130 x0=1300 t0=4.0 half=n inner=n |
	mutter x0=1400 v0=20 t0=5.0 half=n inner=y | 
	mutter v0=2500 x0=1400 t0=5.8 half=n inner=n |
	mutter v0=500 x0=1400 t0=7.0 half=n inner=y  |
	pick rect1=80 rect2=20 vel0=1400
	''')
	
Result('vpick',
       '''
       put d2=1 o2=900 unit2= label2="CMP Number" |
       grey mean=y color=j scalebar=y barreverse=y barunit=m/s
       title="Picked Migration Velocity" label2="CMP Number" unit2= 
       ''')

# Take a slice

Flow('slice','dmo vpick',
     '''
     slice pick=${SOURCES[1]} | 
     put d2=1 o2=900 unit2= label2="CMP Number"
     ''')

Result('slice',
       '''
       window min1=5.5 max1=8.5 | 
       grey title="DMO Stack" 
       ''')

Plot('slice',
     '''
     window min1=5.5 max1=8.5 | 
     grey title="DMO Stack" label2="CMP Number" 
     ''')

# Check one CMP location

p = 380 

min1=5.5
max1=8.0 

Flow('before','stackst',
     '''
     window n3=1 f3=%d min1=%g max1=%g | envelope
     ''' % (p,min1,max1))
Flow('after','envelope',
     '''window n3=1 f3=%d min1=%g max1=%g
     ''' % (p,min1,max1))

for case in ('before','after'):
    Plot(case,
         '''
         grey color=j allpos=y title="%s DMO" 
         label2=Velocity unit2=m/s
         ''' % case.capitalize())
         
Flow('vpick1','vpick',
     '''
     window n2=1 f2=%d min1=%g max1=%g
     '''%(p,min1,max1))

Plot('vpick1',
     '''
      graph yreverse=y transp=y plotcol=7 plotfat=7 
      pad=n min2=%g max2=%g wantaxis=n wanttitle=n
     ''' % (1400,2600))

Plot('before2','before pick','Overlay')
Plot('after2','after vpick1','Overlay')
Result('envelope','before2 after2','SideBySideAniso')

# Stolt Migration 

# 2D cosine transform

Flow('cosft','slice',
     '''
     put d2=16.667 label2=Distance unit2=m | 
     cosft sign1=1 sign2=1 |  
     put label2=Wavenumber
     ''')
Result('cosft','grey title="Cosine Transform" pclip=95')

# Stolt migration with constant velocity

Flow('map2','cosft',
     '''
     math output="sqrt(x1*x1+%g*x2*x2)"
     '''%(562500))

Result('map2',
       '''
       grey color=x allpos=y scalebar=y 
       title="Stolt Map" barlabel=Frequency barunit=Hz
       ''')

Flow('cosft2','cosft map2','iwarp warp=${SOURCES[1]} inv=n')
    
Result('cosft2','grey title="Cosine Transform Warped" pclip=95')

Flow('mig2','cosft2',
     '''
     cosft sign1=-1 sign2=-1 | 
      
     put d2=1 unit2='' 
     ''')

Result('mig2',
       '''
	window min1=5.5 max1=8.5 | 
	grey title="Stolt Migration with 1500 m/s" 
	label2="CMP Number" 
	''')

# Dix conversion to interval velocity
Flow('semb','vscans picks','slice pick=${SOURCES[1]}')
Flow('vdix','picks semb','dix weight=${SOURCES[1]} rect1=50 rect2=10')	

# Stolt Stretch
Flow('cosfts','slice',
     '''
     put d2=16.667 label2=Distance unit2=m | 
     cosft sign2=1 |  
     put label2=Wavenumber
     ''')


# Velocity continuation
Flow('first','slice',
     '''
     put o2=0 d2=16.667 label2=Distance unit2=m o3=0 |
     cosft sign2=1 |
     stolt vel=1400 
     ''')

Flow('mig0','first',
     '''
     window min1=5.5 max1=8.0 | 
     cosft sign2=-1
     ''')

Result('first','mig0',
       '''
       grey title="Migration with 1400 m/s"
       ''')

Flow('velcon','first',
     '''
     vczo nv=101 dv=10 v0=1400 verb=y |
     window min1=5.5 max1=8.0 | 
     transp plane=23 memsize=1000 |
     cosft sign2=-1
     ''')
Result('velcon',
       '''
       byte gainpanel=a | 
       grey3 frame1=300 frame2=200 frame3=50 
       point1=0.8 point2=0.8 flat=n unit3=m/s
       title="Velocity Continuation"
       ''')
Plot('movie','velcon','grey title="velocity continuation"',view=1)

# Migration with the stacking velocity
Flow('mpick','picks','window min1=5.5 max1=8.0')
Flow('mstack','velcon mpick',
     '''
     transp plane=23 memsize=1000 |
     slice pick=${SOURCES[1]}
     ''')
Result('mstack',
       'grey title="Migration with Stacking Velocity"')

# Separating diffractions

Flow('dip','mig0','dip rect1=20 rect2=10 order=2')
Result('dip',
       '''
       grey color=j scalebar=y wanttitle=n barlabel=Slope barunit=samples
       ''')

Flow('refl','mig0 dip','pwspray dip=${SOURCES[1]} ns=5 order=2 reduce=stack')
Result('refl','window min1=5.5 max1=8 | grey title="Reflections" ')

Flow('diff','mig0 refl','add scale=1,-1 ${SOURCES[1]}')
Result('diff','window min1=5.5 max1=8 | grey title="Diffractions" ')

Flow('stoltd','diff',
     '''
     cosft sign2=1 |
     pad beg1=1250
     ''')

Flow('velcond','stoltd',
     '''
     vczo nv=101 dv=10 v0=1400 verb=y pad2=3000 |
     window min1=5.5 max1=8 | 
     transp plane=23 memsize=1000 |
     cosft sign2=-1
     ''')
Plot('velcond','grey title="Velocity Continuation"',view=1)

Flow('focus','velcond',
     '''
     transp plane=23 | 
     pad beg1=1250
     focus rect1=40 rect3=20 |
     window min1=5.5 |
     math output="1/abs(input)" |
     clip clip=8 |
     scale axis=2
     ''')

Flow('fpik','focus','pick v0=1500 rect1=25 rect2=10')

Result('fpik',
       '''
       window | grey color=j allpos=y bias=1500 scalebar=y title="Focusing Velocity"
       label2=CMP barreverse=y barlabel=Velocity barunit=m/s
       ''')

Flow('mdif','velcond fpik',
     '''
     transp plane=23 memsize=1000 |
     slice pick=${SOURCES[1]}
     ''')
Result('mdif',
       'grey title="Focused Diffractions"')

Flow('mfpik','velcon fpik',
     '''
     transp plane=23 memsize=1000 |
     slice pick=${SOURCES[1]}
     ''')
Result('mfpik',
       'grey title="Migration with Focusing Velocity"')

##############################################
# Zoom an interesting area 
##############################################
min1,max1=6.0,6.8 
min2,max2=950,1050
    
for i in range (2):
    case=('slice','mig2')[i]
    zoom = case + '-zoom'
    Flow(zoom,case,
         '''
         window min1=%g max1=%g min2=%g max2=%g
         ''' % (min1,max1,min2,max2))
    Plot(zoom,'grey title=%s grid=y gridcol=5' % ('ab'[i]))
for i in range (2):
    case=('mstack','mfpik')[i]
    zoom2 = case + '-zoom'
    Flow(zoom2,case,
         '''
         put d2=1 o2=900 | window min1=%g max1=%g min2=%g max2=%g
         ''' % (min1,max1,min2,max2))
    Plot(zoom2,'grey title=%s grid=y gridcol=5' % ('cd'[i]))
Result('zoom','slice-zoom mig2-zoom mstack-zoom mfpik-zoom','SideBySideIso')

# Split-step migration
Flow('stackft','stack',
     '''
     put o2=0 d2=16.667 label2=Distance unit2=m o3=0 |
     fft1 | transp plane=12 | transp plane=23 
     ''')

Flow('vdixz','vdix','put o2=0 d2=16.667 d3=16.667 o3=0 | time2depth dz=16.667 intime=y nz=600 intime=y velocity=vdix.rsf')
Flow('slo','vdixz','put o2=0 d2=16.667 d3=16.667 o3=0 | transp | transp plane=23 | math "output=1/input"')
Flow('smig','stackft slo',
     '''
     zomig3 ompnth=1 mode=d --readwrite=y verb=y 
     nrmax=10 slo=${SOURCES[1]} dtmax=0.00005 | 
     zomig3 ompnth=1 mode=m --readwrite=y verb=y 
     nrmax=10 slo=${SOURCES[1]} dtmax=0.00005
     ''',split=[3,'omp',[0]],reduce='add')
Result('smig',
       '''
       window max3=10 | dd type=float | transp | grey title="Split-step Migration" 
       scalebar=y barlabel=Amplitude
       ''')
End()
