---
title: Changing BDA Filter Properties
author: windows-driver-content
description: Changing BDA Filter Properties
ms.assetid: 1833864a-5759-437c-ba60-0b38602d9e41
keywords:
- property sets WDK BDA , filter property changes
- filter property changes WDK BDA
- method sets WDK BDA , filter property changes
ms.author: windowsdriverdev
ms.date: 04/20/2017
ms.topic: article
ms.prod: windows-hardware
ms.technology: windows-devices
---

# Changing BDA Filter Properties


## <a href="" id="ddk-changing-bda-filter-properties-ksg"></a>


Because multiple instances of an application that views media broadcasts can run simultaneously on a system, you should write a BDA minidriver to accommodate multiple instances of a filter. Each filter instance can contain different information. For example, one instance of a tuner filter can contain a request to tune to channel 5, while another instance can contain a request to tune to channel 8. As control transitions from one instance to another, the BDA minidriver must instruct the underlying tuning device to change the way resources are configured. A BDA minidriver processes method requests of the [KSMETHODSETID\_BdaChangeSync](https://msdn.microsoft.com/library/windows/hardware/ff563403) method set to coordinate a list of property requests on one the minidriver's filter instances.

The primary purpose of the KSMETHODSETID\_BdaChangeSync method set is to provide trigger points at which the underlying minidriver for a filter can acquire and release resources from the minidriver's device object. The minidriver must coordinate these trigger points with the filter's transition to and from a stopped state. For example, if the filter is in a stopped state, the minidriver should assign new resources to the filter but not acquire those resources whenever the network provider specifies to commit BDA topology changes. When the filter subsequently transitions out of its stopped state, the minidriver should then attempt to acquire those resources from the underlying device.

On the other hand, if the filter is already active, the minidriver should attempt to acquire new resources from the underlying device whenever the network provider specifies to commit BDA topology changes. Only one instance of the filter can be active--in the running state and holding the same resources--at any given time. When a filter transitions to the stopped state, it should therefore release all its resources, including those resources that were allocated for any of its pins, so that the resources are available to another filter graph that transitions to the running state.

Typically, a BDA minidriver's filter object intercepts and supplies methods for the methods of the KSMETHODSETID\_BdaChangeSync method set. For example, the minidriver supplies methods for starting, checking, and committing filter changes and for getting the change state of a filter. In addition, the following minidriver-supplied methods should call the corresponding BDA support library functions to synchronize changes on the structures that the minidriver previously registered with the BDA support library:

-   Start-changes method calls the [**BdaStartChanges**](https://msdn.microsoft.com/library/windows/hardware/ff556507) function.

-   Check-changes method calls the [**BdaCheckChanges**](https://msdn.microsoft.com/library/windows/hardware/ff556433) function.

-   Commit-changes method calls the [**BdaCommitChanges**](https://msdn.microsoft.com/library/windows/hardware/ff556435) function.

-   Get-changed-state method calls the [**BdaGetChangeState**](https://msdn.microsoft.com/library/windows/hardware/ff556458) function.

The following code snippet shows how to intercept method requests of the KSMETHODSETID\_BdaChangeSync method set using an internal method:

```
//
//  BDA Change Sync Method Set
//
//  Defines the dispatch routines for the filter level
//  Change Sync methods
//
DEFINE_KSMETHOD_TABLE(BdaChangeSyncMethods)
{
    DEFINE_KSMETHOD_ITEM_BDA_START_CHANGES(
        CFilter::StartChanges,
        NULL
        ),
    DEFINE_KSMETHOD_ITEM_BDA_CHECK_CHANGES(
        CFilter::CheckChanges,
        NULL
        ),
    DEFINE_KSMETHOD_ITEM_BDA_COMMIT_CHANGES(
        CFilter::CommitChanges,
        NULL
        ),
    DEFINE_KSMETHOD_ITEM_BDA_GET_CHANGE_STATE(
        CFilter::GetChangeState,
        NULL
        )
};
```

The following code snippet shows how the internal start-changes method in a BDA minidriver resets pending resource changes after the minidriver calls the **BdaStartChanges** support function to initiate the setting of new BDA topology changes:

```
//
// StartChanges ()
//
//    Puts the filter into change state.  All changes to BDA topology
//    and properties changed after this will be in effect only after
//    CommitChanges.
//
NTSTATUS
CFilter::
StartChanges(
    IN PIRP         pIrp,
    IN PKSMETHOD    pKSMethod,
    OPTIONAL PVOID  pvIgnored
    )
{
    NTSTATUS        Status = STATUS_SUCCESS;
    CFilter *       pFilter;

    ASSERT( pIrp);
    ASSERT( pKSMethod);

    // Obtain a "this" pointer for the method.
    //
    // Because this function is called directly from the property 
    // dispatch table, must get pointer to the underlying object.
    //
    pFilter = FilterFromIRP( pIrp);
    ASSERT( pFilter);
    if (!pFilter)
    {
        Status = STATUS_INVALID_PARAMETER;
        goto errExit;
    }

    //  Reset any pending BDA topology changes.
    //
    Status = BdaStartChanges( pIrp);
    if (!NT_SUCCESS( Status))
    {
        goto errExit;
    }

    //  Reset any pending resource changes.
    //
    pFilter->m_NewTunerResource = pFilter->m_CurTunerResource;
    pFilter->m_BdaChangeState = BDA_CHANGES_COMPLETE;

errExit:
    return Status;
}
```

 

 


--------------------
[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Bstream\stream%5D:%20Changing%20BDA%20Filter%20Properties%20%20RELEASE:%20%288/23/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")


