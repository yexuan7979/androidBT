---
description: temp usage
---

# temp

insert code

```text
/*******************************************************************************
**
** Function         bta_av_ci_data
**
** Description      forward the BTA_AV_CI_SRC_DATA_READY_EVT to stream state machine
**
**
** Returns          void
**
*******************************************************************************/
static void bta_av_ci_data(tBTA_AV_DATA *p_data)
{
    tBTA_AV_SCB *p_scb;
    int     i;
    UINT8   chnl = (UINT8)p_data->hdr.layer_specific;
    for( i=0; i < BTA_AV_NUM_STRS; i++ )
    {
        p_scb = bta_av_cb.p_scb[i];
        //Check if the Stream is in Started state before sending data
        //in Dual Handoff mode, get SCB where START is done.
        if(p_scb && (p_scb->chnl == chnl) && (p_scb->started))
        {
            APPL_TRACE_DEBUG("[lily_check]bta_av_ci_data: p_scb 0x%x", p_scb);
            bta_av_ssm_execute(p_scb, BTA_AV_SRC_DATA_READY_EVT, p_data);
        }
    }
}
```



