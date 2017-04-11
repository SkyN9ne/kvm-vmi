# kvm-vmi

KVM-based Virtual Machine Instrospection.

# Description

This project add virtual machine introspection to the KVM hypervisor
to monitor a running virtual machine without a guest agent.

Once the traps are set, the VM will be in a "_paused_" state and go back to the
hypervisor on every system call.
In details, the traps are working directly at the instruction level, on `syscall`
and `sysret`, which means that you can also stop the VM when the system call
returns from the kernel.

When the VM is "_paused_", some introspection can be done by reading or writing
into the memory. Therefore it is possible to reconstruct VM state and understand
the system call `context` (process name, system call name).

Futhermore, we are able to decode the system call
parameters and display what file is being created (in the case of `NtCreateFile`,
for `Windows` only).

A hooking API allows you to define `callbacks` on top of the system calls you intercept:

~~~Python
def enter_NtCreateFile(syscall):
    KeyHandle, DesiredAccess, object_attributes = syscall.collect_args(3)
    obj = ObjectAttributes(object_attributes, syscall.process)
    buffer = obj.ObjectName.Buffer
    access = FileAccessMask(DesiredAccess)
    syscall.hook = {
        'object_name': buffer,
        'access': access.rights
    }
~~~

Resulting in this output:

~~~JSON
[
    {
        "event": {
            "cr3": "0x76f9e000",
            "vcpu": 0,
            "rax": "0x52",
            "direction": "enter",
            "type": "syscall"
        },
        "name": "NtCreateFile",
        "process": {
            "pid": 2344,
            "name": "powershell.exe"
        },
        "hook": {
            "object_name": "\\??\\C:\\Program Files\\Windows Sidebar\\Gadgets\\PicturePuzzle.Gadget\\en-US\\gadget.xml",
            "access": [
                "SYNCHRONIZE",
                "GENERIC_READ",
                "FILE_READ_ATTRIBUTES"
            ]
        }
    },
]
~~~


This project is divided into 2 components:
- a modified version of the KVM source code.
- a userland component which receive and displays events.
- a modified `QEMU`, needed for the memory access

# Setup

## Using Vagrant (recommended)

Go to the `vagrant/` sub-directory to install a development environement for `kvm-vmi`

## Manually

Unfortunately, it is not possible to compile the KVM modules as an `out-of-tree`
build. You will have to compile and install a new kernel along with the new modules.

- Start by compiling a new kernel in `kvm`
- Reboot
- Make sure you loaded the modified kernel module (`make reload`)
- Go to `nitro` to setup the userland component and intercept syscalls
- Compile the modified version of `qemu` if you intend to analyze syscall events


# References

Based on the work of `Jonas Pfoh`:
- [Nitro: Hardware-based System Call Tracing for Virtual Machines](https://www.sec.in.tum.de/assets/staff/pfoh/PfohSchneider2011a.pdf)
- [Nitro - VMI Extensions for Linux/KVM](http://nitro.pfoh.net/)
