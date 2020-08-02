Here are some tips I learnt from ALMA helpdesk. I put them here so it is easier for me to check. ALMA helpdesk helps me quite a lot in understanding ALMA data reduction. I would like to thank Xing Lu, Kazuya Saigo for their patiently help in ALMA helpdesk.

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

