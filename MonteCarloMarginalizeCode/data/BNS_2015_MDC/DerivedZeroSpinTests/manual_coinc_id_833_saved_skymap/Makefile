SHELL=/bin/bash

LIGO_CHANNEL_NAME=GAUSSIAN
#VIRGO_CHANNEL_NAME=h_16384Hz
INJ_CHANNEL_NAME=FAKE-STRAIN
INJ_XML=$(shell if [ -f mdc.xml.gz ]; then echo "mdc.xml.gz" ; else echo "coinc.xml.gz"; fi)
INJECTION_FRAME_TYPE=2015_bns_noise_mdc

INJECTION_DEST=2015_bns_noise_mdc/
MASS1=$(shell if [ ${INJ_XML} == "mdc.xml.gz" ]; then ligolw_print -t sim_inspiral -c mass1 ${INJ_XML}; else ligolw_print -t sngl_inspiral -c mass1 ${INJ_XML}; fi )
MASS2=$(shell if [ ${INJ_XML} == "mdc.xml.gz" ]; then ligolw_print -t sim_inspiral -c mass2 ${INJ_XML}; else ligolw_print -t sngl_inspiral -c mass2 ${INJ_XML}; fi )
#EVENT_TIME=
#EVENT_TIME=`ligolw_print -c geocent_end_time -c geocent_end_time_ns -d "." ${INJ_XML}`
EVENT_TIME_S=$(shell if [ ${INJ_XML} == "mdc.xml.gz" ]; then ligolw_print -c geocent_end_time ${INJ_XML}; else ligolw_print -c end_time ${INJ_XML}; fi )
EVENT_TIME_NS=$(shell if [ ${INJ_XML} == "mdc.xml.gz" ]; then ligolw_print -c geocent_end_time_ns ${INJ_XML}; else ligolw_print -c end_time_ns ${INJ_XML}; fi )
EVENT_TIME=$(shell printf "%d.%09d" ${EVENT_TIME_S} ${EVENT_TIME_NS})

START=$(shell python -c "print int(${EVENT_TIME})-200")
STOP=$(shell python -c "print int(${EVENT_TIME})+200")
DUR=$(shell python -c "print ${STOP}-${START}")

COINC_ID=$(shell ligolw_print -t coinc_inspiral -c coinc_event_id ${INJ_XML} )
COINC_ID_INT=$(shell echo ${COINC_ID} | awk -F ':' '{print $$3}')

LIGO_FRAME_DURATION=64 # s
LIGO_FRAME_DURATION_COMB=64 # s
LIGO_N_FRAMES=1

print_event_time:
	@echo ${EVENT_TIME}

print_coinc:
	@echo ${COINC_ID_INT}

#postprocess: neff_evid.png logevidence_hist.png
postprocess:
	mkdir xml sqlite plots
	mv 2015_bns_*.xml.gz xml/
	mv ILE*.sqlite sqlite/
	mv ILE*.png plots/
	rm ILE*.cache

archive:
	tar zcf run_logs.tgz logs/* 
	rm -rf logs/*

adapt_beta.txt: ${INJ_XML}
	if [ ${INJ_XML} == "mdc.xml.gz" ]; then util_AdaptiveExponent.py --inj ${INJ_XML} --channel-name "H1:FAKE-STRAIN" --channel-name "L1:FAKE-STRAIN" --psd-file-singleifo "H1=H1_PSD_measured.xml.gz" --psd-file-singleifo "L1=L1_PSD_measured.xml.gz" --Nskip 1000 2> /dev/null | tail -1 > adapt_beta.txt; else util_AdaptiveExponent.py --coinc ${INJ_XML} --Nskip 1000 | tail -1 > adapt_beta.txt; fi

HL-FAKE_SEARCH.xml.gz: mdc.xml.gz H1_PSD_measured.xml.gz L1_PSD_measured.xml.gz
	~/excesspower/excesspower-utils/bin/ligolw_inj_snr  \
		--det-psd-func L1=L1_PSD_measured.xml.gz \
		--det-psd-func H1=H1_PSD_measured.xml.gz \
		--calculate-only \
		--store-sngl sngl_inspiral \
		mdc.xml.gz

neff_evid.png: HL-FAKE_SEARCH.xml.gz ILE_all.cache
	../harvest_neff HL-FAKE_SEARCH.xml.gz logs/

logevidence_hist.png: ILE_all.cache HL-FAKE_SEARCH.xml.gz
	../followup ILE_all.cache HL-FAKE_SEARCH.xml.gz

ILE_all.cache:
	find sqlite -name "*.sqlite" | lalapps_path2cache -a > ILE_all.cache

marginalize_extrinsic_parameters.dag: 2015_bns_mdc.cache H1_PSD_measured.xml.gz L1_PSD_measured.xml.gz adapt_beta.txt #${COINC_ID_INT}.toa_phoa_snr.fits.gz
	compute_marginalized_likelihood --cache-file 2015_bns_mdc.cache --channel-name H1=${INJ_CHANNEL_NAME} --channel-name L1=${INJ_CHANNEL_NAME} --psd-file "H1=H1_PSD_measured.xml.gz" --psd-file "L1=L1_PSD_measured.xml.gz" --event-time ${EVENT_TIME} --mass1 ${MASS1} --mass2 ${MASS2} --reference-freq 0.0 --save-samples --output-file 2015_bns_noise.xml.gz --n-copies 10 --save-P 1e-4 --fmax 2000 --n-chunk 1000 --time-marginalization --adapt-weight-exponent `cat adapt_beta.txt` --linear-spoked --match-value 0.90 --skymap-file ${COINC_ID_INT}.toa_phoa_snr.fits.gz

test_ile_sim_only_noise: 2015_bns_noise.cache H1_PSD_measured.xml.gz L1_PSD_measured.xml.gz
	integrate_likelihood_extrinsic --cache-file 2015_bns_noise.cache --channel-name H1=${LIGO_CHANNEL_NAME} --channel-name L1=${LIGO_CHANNEL_NAME} --psd-file "H1=H1_PSD_measured.xml.gz" --psd-file "L1=L1_PSD_measured.xml.gz" --event-time ${EVENT_TIME} --mass1 ${MASS1} --mass2 ${MASS2} --pin-to-sim ${INJ_XML} --reference-freq 0.0 --fmax 2000

test_ile: 2015_bns_mdc.cache H1_PSD_measured.xml.gz L1_PSD_measured.xml.gz
	integrate_likelihood_extrinsic --cache-file 2015_bns_mdc.cache --channel-name H1=${INJ_CHANNEL_NAME} --channel-name L1=${INJ_CHANNEL_NAME} --psd-file "H1=H1_PSD_measured.xml.gz" --psd-file "L1=L1_PSD_measured.xml.gz" --event-time ${EVENT_TIME} --mass1 ${MASS1} --mass2 ${MASS2} --pin-to-sim ${INJ_XML} --reference-freq 0.0 --fmax 2000

psds: H1_PSD_measured.xml.gz L1_PSD_measured.xml.gz

plots: H1_PSD_measured.png L1_PSD_measured.png
	../../utils/plot_psds {H,L}1_PSD_measured.xml.gz

2015_bns_mdc.cache: H1_combined.cache L1_combined.cache
	cat {H,L}1_combined.cache > 2015_bns_mdc.cache

H1_combined.cache: 2015_bns_data.cache
	gstlal_fake_frames --gps-start-time ${START} --gps-end-time ${STOP} --channel H1=${LIGO_CHANNEL_NAME} --verbose --data-source frames --frame-cache 2015_bns_data.cache --output-channel ${INJ_CHANNEL_NAME} --frame-type ${INJECTION_FRAME_TYPE} --output-path ${INJECTION_DEST} --frame-duration ${LIGO_FRAME_DURATION} --frames-per-file ${LIGO_N_FRAMES} --injections ${INJ_XML}
	find ${INJECTION_DEST}/ -name "H-*.gwf" | lalapps_path2cache > H1_combined.cache

L1_combined.cache: 2015_bns_data.cache
	gstlal_fake_frames --gps-start-time ${START} --gps-end-time ${STOP} --channel L1=${LIGO_CHANNEL_NAME} --verbose --data-source frames --frame-cache 2015_bns_data.cache --output-channel ${INJ_CHANNEL_NAME} --frame-type ${INJECTION_FRAME_TYPE} --output-path ${INJECTION_DEST} --frame-duration ${LIGO_FRAME_DURATION} --frames-per-file ${LIGO_N_FRAMES} --injections ${INJ_XML}
	find ${INJECTION_DEST}/ -name "L-*.gwf" | lalapps_path2cache > L1_combined.cache

H1_PSD_measured.xml.gz: 2015_bns_data.cache
	gstlal_reference_psd --gps-start-time ${START} --gps-end-time ${STOP} --channel H1=${LIGO_CHANNEL_NAME} --verbose --data-source frames --frame-cache 2015_bns_data.cache --write-psd H1_PSD_measured.xml.gz

H1_PSD_measured.png: H1_PSD_measured.xml.gz
	../utils/plot_psds_single H1 H1_PSD_measured.xml.gz ../HLV-ILIGO_PSD.xml.gz
	mv psd_single.png H1_PSD_measured.png

L1_PSD_measured.xml.gz: 2015_bns_data.cache
	gstlal_reference_psd --gps-start-time ${START} --gps-end-time ${STOP} --channel L1=${LIGO_CHANNEL_NAME} --verbose --data-source frames --frame-cache 2015_bns_data.cache --write-psd L1_PSD_measured.xml.gz

L1_PSD_measured.png: L1_PSD_measured.xml.gz
	../utils/plot_psds_single L1 L1_PSD_measured.xml.gz ../HLV-ILIGO_PSD.xml.gz
	mv psd_single.png L1_PSD_measured.png

clean:
	rm {H,L,V}1*.xml.gz
	rm -rf ${INJECTION_DEST}
	rm *.cache
	rm *.png

2015_bns_data.cache:
	ligo_data_find --gaps -o L -l -t L1_T1200307_V4_EARLY_GAUSSIAN -u file -s ${START} -e ${STOP} > 2015_bns_data.cache
	ligo_data_find --gaps -o H -l -t H1_T1200307_V4_EARLY_GAUSSIAN -u file -s ${START} -e ${STOP} >> 2015_bns_data.cache
