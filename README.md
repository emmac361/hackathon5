    #this is the PCR lyticase lysis buffer part 

    from opentrons import simulate
    metadata = {'apiLevel': '2.0'}
    protocol = simulate.get_protocol_api('2.0')
    #Labware
    plate = protocol.load_labware('corning_96_wellplate_360ul_flat', 1)
    tiprack_1 = protocol.load_labware('opentrons_96_tiprack_300ul', 2)
    #pipettes
    p300 = protocol.load_instrument('p300_single_gen2', 'right', tip_racks=[tiprack_1])
    protocol.max_speeds['Z'] = 10
    
    #commands
    p300.transfer(50, plate['A1'], plate['C1'])
    p300.transfer(5, plate['B1'], plate['C1'])

    p300.transfer(55, plate['C1'], plate['D1'], touch_tip=True, blow_out=True, new_tip='always') 
    p300.pick_up_tip()


    #mix(repetitions, volume, location, rate)
    p300.mix(5, 55, plate['D1'], 0.5)
    p300.return_tip()

    #thermocycler part/or use temperature module
    #first incubation time(37˙C for 30 minutes)
    #second incubation time(95˙C for 10 minutes)
    protocol.delay(minutes=40)   

    p300.transfer(5, plate['D1'], plate['A2'])

    #pause for 5 minutes to prepare for PCR/thermocycler reaction
    protocol.delay(minutes=5)           
    
    for line in protocol.commands(): 
        print(line)

    #this is the PCR reaction
    def get_values(*names):
    import json
    _all_values = json.loads("""{"number_of_samples":96,"dna_volume":13.4,"mastermix_volume":4.6,"master_mix_csv":"Reagent,Well,Volume\\nBuffer,A2,3\\nMgCl,A3,40\\ndNTPs,A2,90\\nWater,A3,248\\nprimer 1,A4,25\\nprimer 2,A5,25\\n","tuberack_type":"opentrons_24_aluminumblock_nest_1.5ml_screwcap","single_channel_type":"p1000_single_gen2","single_channel_mount":"right","pipette_2_type":"p1000_single_gen2","pipette_2_mount":"left","lid_temp":105,"init_temp":96,"init_time":30,"d_temp":96,"d_time":15,"a_temp":60,"a_time":30,"e_temp":74,"e_time":30,"no_cycles":30,"fe_temp":74,"fe_time":30,"final_temp":4}""")
    return [_all_values[n] for n in names]


    import math

    # metadata
    metadata = {
    'protocolName': 'Complete PCR Workflow with Thermocycler',
    'author': 'Nick <protocols@opentrons.com>',
    'source': 'Custom Protocol Request',
    'apiLevel': '2.0'
    }
    def run(ctx):

    [number_of_samples, dna_volume, mastermix_volume,
     master_mix_csv, tuberack_type, single_channel_type, single_channel_mount,
     pipette_2_type, pipette_2_mount, lid_temp, init_temp, init_time, d_temp,
     d_time, a_temp, a_time, e_temp, e_time, no_cycles, fe_temp, fe_time,
     final_temp] = get_values(  # noqa: F821
        'number_of_samples', 'dna_volume', 'mastermix_volume',
        'master_mix_csv', 'tuberack_type', 'single_channel_type',
        'single_channel_mount', 'pipette_2_type', 'pipette_2_mount',
        'lid_temp', 'init_temp', 'init_time', 'd_temp', 'd_time', 'a_temp',
        'a_time', 'e_temp', 'e_time', 'no_cycles', 'fe_temp', 'fe_time',
        'final_temp')

    range1 = single_channel_type.split('_')[0][1:]
    tipracks1 = [
        ctx.load_labware('opentrons_96_tiprack_' + range1 + 'ul', slot)
        for slot in ['2', '3']
    ]
    p1 = ctx.load_instrument(
        single_channel_type, single_channel_mount, tip_racks=tipracks1)

    using_multi = True if pipette_2_type.split('_')[1] == 'multi' else False
    if using_multi:
        mm_plate = ctx.load_labware(
            'nest_96_wellplate_100ul_pcr_full_skirt', '4',
            'plate for mastermix distribution')
    if pipette_2_type and pipette_2_mount:
        range2 = pipette_2_type.split('_')[0][1:]
        tipracks2 = [
            ctx.load_labware('opentrons_96_tiprack_' + range2 + 'ul', slot)
            for slot in ['6', '9']
        ]
        p2 = ctx.load_instrument(
            pipette_2_type, pipette_2_mount, tip_racks=tipracks2)

    # labware setup
    tc = ctx.load_module('thermocycler')
    tc_plate = tc.load_labware(
        'nest_96_wellplate_100ul_pcr_full_skirt', 'thermocycler plate')
    if tc.lid_position != 'open':
        tc.open_lid()
    tc.set_lid_temperature(lid_temp)
    if 'cooled' in tuberack_type:
        tempdeck = ctx.load_module('tempdeck', '1')
        tuberack = tempdeck.load_labware(
            tuberack_type, 'rack for mastermix reagents'
        )
    else:
        tuberack = ctx.load_labware(
            tuberack_type, '1', 'rack for mastermix reagents')
    dna_plate = ctx.load_labware(
        'nest_96_wellplate_100ul_pcr_full_skirt', '5', 'DNA plate')
    
      # reagent setup
    mm_tube = tuberack.wells()[0]
    num_cols = math.ceil(number_of_samples/8)

    pip_counts = {p1: 0, p2: 0}
    p1_max = len(tipracks1)*96
    p2_max = len(tipracks2)*12 if using_multi else len(tipracks2)*96
    pip_maxs = {p1: p1_max, p2: p2_max}

    def pick_up(pip):
        if pip_counts[pip] == pip_maxs[pip]:
            ctx.pause('Replace empty tipracks before resuming.')
            pip.reset_tipracks()
            pip_counts[pip] = 0
        pip.pick_up_tip()
        pip_counts[pip] += 1

    # determine which pipette has the smaller volume range
    if using_multi:
        pip_s, pip_l = p1, p1
    else:
        if int(range1) <= int(range2):
            pip_s, pip_l = p1, p2
        else:
            pip_s, pip_l = p2, p1

    # destination
    mastermix_dest = tuberack.wells()[0]

    info_list = [
        [cell.strip() for cell in line.split(',')]
        for line in master_mix_csv.splitlines()[1:] if line
    ]

    """ create mastermix """
    for line in info_list[1:]:
        source = tuberack.wells(line[1].upper())
        vol = float(line[2])
        pip = pip_s if vol <= pip_s.max_volume else pip_l
        pick_up(pip)
        pip.transfer(vol, source, mastermix_dest, new_tip='never')
        pip.drop_tip()

    """ distribute mastermix and transfer sample """
    if tc.lid_position != 'open':
        tc.open_lid()
    if using_multi:
        mm_source = mm_plate.rows()[0][0]
        mm_dests = tc_plate.rows()[0][:num_cols]
        vol_per_well = mastermix_volume*num_cols*1.05
        pick_up(p1)
        for well in mm_plate.columns()[0]:
            p1.transfer(vol_per_well, mm_tube, well, new_tip='never')
            p1.blow_out(well.top(-2))
        p1.drop_tip()
        pip_mm = p2

    else:
        mm_source = mm_tube
        mm_dests = tc_plate.wells()[:number_of_samples]
        pip_mm = pip_s if mastermix_volume <= pip_s.max_volume else pip_l

    for d in mm_dests:
        pick_up(pip_mm)
        pip_mm.transfer(mastermix_volume, mm_source, d, new_tip='never')
        pip_mm.drop_tip()

    # transfer DNA to corresponding well
    if using_multi:
        dna_sources = dna_plate.rows()[0][:num_cols]
        dna_dests = tc_plate.rows()[0][:num_cols]
        pip_dna = p2
    else:
        dna_sources = dna_plate.wells()[:number_of_samples]
        dna_dests = tc_plate.wells()[:number_of_samples]
        pip_dna = pip_s if dna_volume <= pip_s.max_volume else pip_l

    for s, d in zip(dna_sources, dna_dests):
        pick_up(pip_dna)
        pip_dna.transfer(
            dna_volume, s, d, mix_after=(5, 0.8*mastermix_volume + dna_volume),
            new_tip='never')
        pip_dna.drop_tip()

    """ run PCR profile on thermocycler """

    # Close lid
    if tc.lid_position != 'closed':
        tc.close_lid()

    # lid temperature set
    tc.set_lid_temperature(lid_temp)

    # initialization
    well_vol = mastermix_volume + dna_volume
    tc.set_block_temperature(
        init_temp, hold_time_seconds=init_time, block_max_volume=well_vol)

    # run profile
    profile = [
        {'temperature': d_temp, 'hold_time_seconds': d_time},
        {'temperature': a_temp, 'hold_time_seconds': a_temp},
        {'temperature': e_temp, 'hold_time_seconds': e_time}
    ]


    tc.execute_profile(
        steps=profile, repetitions=no_cycles, block_max_volume=well_vol)

    # final elongation
    tc.set_block_temperature(
        fe_temp, hold_time_seconds=fe_time, block_max_volume=well_vol)

    # final hold
    tc.deactivate_lid()
    tc.set_block_temperature(final_temp)
 
    from opentrons.simulate import simulate, format_runlog
    protocol_file = open(r'C:\Users\arman\Downloads\complete-pcr.py')
    runlog, _bundle = simulate(protocol_file)
    print(format_runlog(runlog))

    #this is the thermocycler bit


    def get_values(*names):
    import json
    _all_values = json.loads("""{"well_vol":50,"lid_temp":110,"init_temp":4,"init_temp1":98,"init_time1":30,"init_time":300, "d_temp":98,"d_time":10,"a_temp":60,"a_time":20,"e_temp":72,"e_time":60,"no_cycles":8,"d_temp1":98,"d_time1":10,"a_temp1":52.4,"a_time1":20,"e_temp1":72,"e_time1":90,"no_cycles1":25,"fe_temp":72,"fe_time":600,"final_temp":4}""")
    return [_all_values[n] for n in names]


    metadata = {
    'protocolName': 'Thermocycler Example Protocol',
    'author': 'Opentrons <protocols@opentrons.com>',
    'source': 'Protocol Library',
    'apiLevel': '2.0'
    }


    def run(protocol):
    [well_vol, lid_temp, init_temp, init_time,
        d_temp, d_time, a_temp, a_time,
        e_temp, e_time, no_cycles,
        fe_temp, fe_time, final_temp] = get_values(  # noqa: F821
    'well_vol', 'lid_temp', 'init_temp', 'init_time', 'd_temp', 'd_time',
        'a_temp', 'a_time', 'e_temp', 'e_time', 'no_cycles',
        'fe_temp', 'fe_time', 'final_temp')

    tc_mod = protocol.load_module('thermocycler')

    """
    Add liquid transfers here, if interested (make sure TC lid is open)
    Example (Transfer 50ul of Sample from plate to Thermocycler):

    tips = [protocol.load_labware('opentrons_96_tiprack_300ul', '2')]
    pipette = protocol.load_instrument('p300_single', 'right', tip_racks=tips)
    tc_plate = tc_mod.load_labware('nest_96_wellplate_100ul_pcr_full_skirt')
    sample_plate = protocol.load_labware('nest_96_wellplate_200ul_flat', '1')

    tc_wells = tc_plate.wells()
    sample_wells = sample_plate.wells()

    if tc_mod.lid_position != 'open':
        tc_mod.open_lid()

    for t, s in zip(tc_wells, sample_wells):
        pipette.transfer(50, s, t)
    """
    # setting to 4 degress for the begginging for adding things
    tc_mod.set_block_temperature(init_temp, hold_time_seconds=init_time,
                                 block_max_volume=well_vol)
    
    # Close lid
    if tc_mod.lid_position != 'closed':
        tc_mod.close_lid()

    # lid temperature 
    tc_mod.set_lid_temperature(lid_temp)

    # set to 98 degrees for 30 seconds
    tc_mod.set_block_temperature(init_temp1, hold_time_seconds=init_time1,
                                 block_max_volume=well_vol)
    
    # Run first cycle total 720 seconds
    profile = [
        {'temperature': d_temp, 'hold_time_seconds': d_time},
        {'temperature': a_temp, 'hold_time_seconds': a_temp},
        {'temperature': e_temp, 'hold_time_seconds': e_time}
    ]

    tc_mod.execute_profile(steps=profile, repetitions=no_cycles,
                           block_max_volume=well_vol)
    # Run second cycle total 3000 seconds
    profile = [
        {'temperature': d_temp1, 'hold_time_seconds': d_time1},
        {'temperature': a_temp1, 'hold_time_seconds': a_temp1},
        {'temperature': e_temp1, 'hold_time_seconds': e_time1}
    ]

    tc_mod.execute_profile(steps=profile, repetitions=no_cycles,
                           block_max_volume=well_vol)
    
    
    # at 72 degrees for ten mins

    tc_mod.set_block_temperature(fe_temp, hold_time_seconds=fe_time,
                                 block_max_volume=well_vol)

    # holding at 4 degrees (protocol said 10 but 4 degrees much better)
    tc_mod.deactivate_lid()
    tc_mod.set_block_temperature(final_temp)
