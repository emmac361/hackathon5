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

#first incubation time(37˙C for 30 minutes)
#second incubation time(95˙C for 10 minutes)
protocol.delay(minutes=40)   

p300.transfer(5, plate['D1'], plate['A2'])

#pause for 5 minutes to prepare for PCR/thermocycler reaction
protocol.delay(minutes=5)           
    
for line in protocol.commands(): 
        print(line)


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
