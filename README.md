Here are some tips I learnt from ALMA helpdesk and somewhere else. I put them here so it is easier for me to check. ALMA helpdesk helps me quite a lot in understanding ALMA data reduction. I would like to thank Edo Ibar, Xing Lu, Kazuya Saigo, Atsushi Miyazaki, Chentao Yang for their patiently help.

## 0， About Jy/Beam

The total flux within specified area for extended component are estimated from the flux densities each pixel as follows,

Flux[Jy] = Σ I[Jy/beam] dΩ/Ωbeam

where dΩ and Ωbeam are,

Ωbeam = 2 pi σmaj*σmin = pi/(4 ln 2) * FWHMmaj*FWHMmin = 1.133 * FWHMmaj*FWHMmin # Beam Area

(FWHMmaj,FWHMmin : major/minor axes (FWHM) of Gaussian beam)

dΩ = (pixel size)^2 # Area of a pixel

Thus the equation is changed,

Flux[Jy] = Σ I[Jy/beam] * (4 ln 2)/pi * (pixel size)^2/(FWHMmaj*FWHMmin)

dΩ/Ωbeam means the ratio between the pixel area and the beam area, and is equivalent with the pixel number within one beam. Then it means the conversion from Jy/beam to Jy/pixel. The total flux [Jy] is estimated to sum up the pixel value I[Jy/pixel] within specified area.

* From JCMT webpage: https://www.eaobservatory.org/jcmt/faq/how-can-i-convert-from-mjybeam-to-mjy/

A common question we receive here at the JCMT is regarding observations with SCUBA-2 and the conversion from mJy/beam to mJy. Before we begin, *it should be noted that for a real point source, a peak brightness value reported in units of mJy is the same as a peak brightness value reported in mJy/beam.*

Now what happens if we have a map in mJy/beam and we want to obtain an integrated intensity value, a total flux value. We first sum up a number of pixels and now we want to get our units correct from mJy/beam to mJy…

> Total Flux = flux summed over a number of pixels/(number of pixels in a beam)

Then your units are:

> [mJy*pixels/beam] / [pixels/beam] = [mJy].

* From Edo:

For the point source, the total flux equals to the peak value in the unit of Jy/Beam, because:

S[Jy] = Peak[Jy/Beam] \times A_source / A_beam

For an extended source, the source area is the area of photometry aperture. The beam size is A_beam = 1.13309 * BMAJ * BMIN

in short, for one pixel, the flux density[Jy] = flux[Jy/Beam]/N_beam, where N_beam is the pixel number in one beam.

## 1, How to substract one target from uv plane?

This question comes from the case when I need to draw the uv-amp plot of one target, but the ALMA FoV happen to have two targets, so the uv-amp plot based on the .ms file would include the flux from the two targets. Then I need to substract the target I do not need. Here is the process:

1, tclean the xx.ms file and set savemodel = 'modelcolumn' in tclean, then there would be the two targets in the image, waiting to be cleaned. Then I clean one target only, the model of this target will be saved in the xx.ms file.

2, uvsub('xx.ms', reverse=False) now the model of one target would be subtract from the xx.ms file. Now the xx.ms file have only one target. The other target has only residual.

To check this result, I tclean xx.ms again, and found there is indeed only one target in the image.

## 2, How to combine the two .ms files together to get the .ms file with only one field ID, so that I can use uvmodelfit

The motivition of combine the two .ms files together is to use the uvmodelfit, which works to one field ID. So I have to combine .ms files together, and get only one field ID.

concat in CASA can combine the two .ms files together. If the phasecenter in the two .ms files are not quite the same, the parameter dirtol can help to identify the same target. I tried to concat the .ms files of ID 141 in different cycles. It works.

If I need to unify the phasecenter, I can use fixvis or mstransform to do this. It works.

If somehow I need to change the field name, here is the way: https://casaguides.nrao.edu/index.php/Renaming_a_Field It works.

But for some reason I am not sure, when the two .ms file have identical field ID, observation ID or phasecenter ra, dec, or other parameters, the combineed .ms file may still have two field IDs, with the same ra, dec, then the uvmodelfit still fit only one field ID. The problem may be caused by the split, that the split may not split the files because the values are identical. Here is the way to combine the two:

Here xx.ms is the data combined by concat, I average the channels of the ms file following the suggestion from uvmodelfit here: https://casa.nrao.edu/casadocs/casa-6.0/calibration-and-visibility-data/uv-manipulation/fitting-gaussians-to-visibilities 

    split('xx.ms','xxxx.ms', datacolumn='all',field='*',width='64')

    os.system('rm -rf xxx_cycle2.ms')

    os.system('rm -rf xxx_cycle3.ms')

    split(vis='xxxx.ms', observation='0',datacolumn='all', outputvis='xxx_cycle2.ms')

    split(vis='xxxx.ms', observation='1~5',datacolumn='all', outputvis='xxx_cycle3.ms')

For cycle 2 data, there is a a script download here: https://help.almascience.org/index.php?/Knowledgebase/Article/View/352 
In general, cycle2 data use J2000 which should be unified to ICRS. So I download the relabelmstoicrs.py from this webpage, and replace the file name by the xx.ms, then run it in casa execfile('relabelmstoicrs.py')

Next step is tricky: the two .ms file from cycle 2 and cycle 3 are *NOT splitted well*, maybe because the field name, ra, dec are the same, so we have to remove the other field by hand. This might be a bug of split.

    msname = 'xxx_cycle2.ms'
  
    tb.open(msname+'/FIELD', nomodify=False)

    tb.removerows(1) # The second row in the FIELD table is the ICRS field in the original field (from cycle 3) and we need to delete it.

    tb.close() # Remember to close the table!
    
    msname = 'xxx_cycle3.ms' #
    
    tb.open(msname+'/FIELD', nomodify=False)

    tb.removerows(0) # The first row in the FIELD table is the J2000 field in the original field (from cycle 2) and we need to delete it.
    
    tb.close()

Then the two ms files have only one field, and can be combined by concat:

    os.system('rm -rf S11_avg_cont.ms')
    
    concat(vis=['xxx_cycle2.ms','xxx_cycle3.ms'], concatvis='xxx_cont.ms')

Now there is only one field in xxx_cont.ms, with all the nRows there. Then it is ok to use uvmodelfit.

## 3, How to get a circle beam?

The beam shape is the result of the uv coverage, and the beam is usually not a round circle.

Sometimes one may would like to make a circling beam to have a better comparison with the optical images. 

One method is to convolve the image with an elliptical gaussian kernel, orthogonal to the current beam shape. But this method is not recommended by the ALMA helpdesk. I am asking why, but no reply yet. A similar result would get by setting restoringbeam =1.0arcsec in tclean.

Another method is uvtaper in tclean. Set

        uvtaper = ['1.arcsec','1.arcsec','0deg'], 

in tclean, then the final image would have a rougly circle beam. The difference between restoringbeam and uvtaper is that restoringbeam will smooth the image after clean, while uvtaper will ``smooth'' the data before clean. So restoringbeam will convolve the beam into round shape. uvtaper will change the weight of the values in uv plane, and would not always produce round beam. If uvtaper and restoringbeam are set, the result would be more like the cleaned image and convolved into round beam.

>The uvtaper controls just weight of visibilities in the uv-plane, and it is not ensured to obtain the synthesized beam as the same size as the gaussian shape defined in the uvtaper parameter. This weighting scheme with the uvtaper gives more weight to short baselines and degrades angular resolution, and thus may improve sensitivity for extended structure sampled by short baselines.

>Although, in the case of data with ideal uv-coverage and very high SNR, the images for different weighting should be consistent, the images which is not exactly consistent may be produced in the actual data because sensitivity is affected by different weighting in each baseline length.

>However, if flagging of a few antenna-baseline or uvtaper with a gaussian shape which is not widely different from the native synthesized beam, I think the difference of the produced images probably is not so serious.

>As for the restoringbeam parameter in tclean, according to CASA docs, the final CLEANed image is produced to be deconvolved from the clean model and the gaussian restoring beam defined by the restoringbeam parameter instead of the PSF calculated from native dirty beam. This is similar to spatial gaussian smoothing. However, this parameter may be not recommended. A CASA memo was released about the restoring beam and please check that also.

>> https://casa.nrao.edu/casadocs/casa-6.0/memo-series/casa-memos/
>> CASA Memo 10: Restoring/Common Beam and Flux Scaling in tclean

> It's difficult issue to adjust synthesized beam, and thus it's recommended that the you carefully assess the results compared some ways.

## 4, How to read the FIELD names from ms file?

use tb.open and then combine them like this:

    msfilelist = ['pointing1.ms', 'pointing2.ms', ..., 'pointingN.ms']

    fieldlist = np.empty(shape=(0,))

    for ms in msfilelist:

        print(ms)
    
        tb.open(ms+'/FIELD/')
    
        fieldlist = np.append(fieldlist,tb.getcol('NAME'))
    
        print(fieldlist)
    
        tb.close() 

    print(fieldlist.shape, np.unique(fieldlist).shape)

    field_id = np.unique(fieldlist)


or if I have enough disk space, I can also combine the ms files together by concat, and msmd.open(msfile) it.


## 5, How to read the SPW channel number from ms file?

    tb.open(ms+'/DATA_DESCRIPTION/')
    
    spw_id = tb.getcol('SPECTRAL_WINDOW_ID')
    
    tb.open(ms+'/SPECTRAL_WINDOW/')
    
    num_chan = tb.getcol('NUM_CHAN')
    
    tb.close()
    
tb.open can get the values of ms file: https://casa.nrao.edu/casadocs/casa-5.4.1/reference-material/measurement-set

## 6, How to estimate the beam size of the MS file from CASA

Patricio kindly offered me several methods to estimate the beam size. I found the EstimateSynthesizedBeam function works good for me to have a rough idea about the beam size.

    Dear Cheng,

    I can come up with two ways to estimate the synthesized beam before imaging.

    1.- You can get the synthesized beam from the archive.

    2.- You could install the analysisUtils from NRAO. These tools provide a single command to estimate the angular resolution from a visibility file. The instructions for installation are here.

    https://casaguides.nrao.edu/index.php?title=Analysis_Utilities

    The list of commands with the description are here (please note that not all of them have details publicly available.)

    https://safe.nrao.edu/wiki/bin/view/Main/CasaExtensions

    The command you need would be this one (which I think it is only an estimation because your resolution will depend on the parameters you use during cleaning):

    https://safe.nrao.edu/wiki/bin/view/ALMA/EstimateSynthesizedBeam

    In my experience, I always have to make a dirty image to fine tune the best pixel size, but the above ways can give you the first approximation.

    Cheers,

    Patricio

## 7, How to obtain the calibrated ms file?

1, run script_for_pi.py in the CASA with recommended version in the QA report, or readme.txt. For the CASA version above 4.2.2, it works good.

2, EA, EU and US ARC can provide calibrated measurement sets obtained from script_for_pi.py .

We can either submit a ticket to ALMA helpdesk: https://www2.nao.ac.jp/~eaarc/DATARED/calibrated_ms_request.html

Or get the science ready data products from NRAO: https://data.nrao.edu/portal/#/ 

The above services from EU, EA, NA only run the script_for_pi.py in the script folder. So if the data request some special treatments. 

Example 1: if we run script_for_PI.py of 2015.1.00456.S then we will get only stripe pattern and no target in the image because in readme file: "some of the far-out antennas showed significant short term phase fluctuations", and in the imaging script: "some antennas are located on very remote pads, and were kept for calibration purposes." so we need handle care.

Example 2: 2013.1.01358.S data is obtained in early cycles, and the calibration process in the last part of script_for_pi.py is commented out. And it is possible to add them back by runing the calibration process.
