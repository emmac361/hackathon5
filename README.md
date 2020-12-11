

#this is the thermocycler bit. 


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
