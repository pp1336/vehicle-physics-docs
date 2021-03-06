# Custom Vehicle Controllers

Vehicle Physics Pro includes [VPVehicleController](../components/vehicle-controller.md), a
full-featured vehicle controller ready for simulating most types of vehicles. If you need a kind of
vehicle not supported by VPVehicleBase then you can write your own vehicle controller easily.

A vehicle controller derives from `VehiclePhysics.VehicleBase`. It implements the logic of the
vehicle by overriding the virtual methods and instancing and connecting the blocks representing the
internal parts of the vehicle.

The virtual methods and their roles are detailed at [VehicleBase reference](vehiclebase-reference.md).

## Example source code

This is the minimum code required for implementing a custom 4-wheeled vehicle controller with
steering and rear-powered wheels in Vehicle Physics Pro.

The drive power is provided by a direct drive motor (think on an ideal electric engine) that
provides up to a maximum torque (maxDriveTorque) and can reach a maximum RPMs (maxDriveRpm). Rear
wheels are connected to the direct drive with a differential in the default configuration (Open).

This example doesn't make use of the [data bus](databus-reference.md). Instead, it exposes the
properties driveInput, brakeInput and steerInput. The SimpleVehicleControllerInput.cs script reads
the standard Unity Input and modifies these properties for controlling the vehicle.


**SimpleVehicleController.cs**
```
using UnityEngine;
using VehiclePhysics;

public class SimpleVehicleController : VehicleBase
	{
	[Header("Simple Vehicle Controller")]
	public VPWheelCollider wheelFL;
	public VPWheelCollider wheelFR;
	public VPWheelCollider wheelRL;
	public VPWheelCollider wheelRR;
	public TireFriction tireFriction = new TireFriction();

	public float maxDriveTorque = 500.0f;
	public float maxBrakeTorque = 1000.0f;
	public float maxSteerAngle = 45.0f;
	public float maxDriveRpm = 50.0f;

	[Range(-1,1)]
	public float driveInput = 0.0f;
	[Range(0,1)]
	public float brakeInput = 0.0f;
	[Range(-1,1)]
	public float steerInput = 0.0f;

	// Internal vehicle blocks (powertrain)

	DirectDrive m_directDrive;
	Differential m_differential;

	// Initialize the controller and blocks

	protected override void OnInitialize ()
		{
		// Declare the number of wheels

		SetNumberOfWheels(4);

		if (wheelFL == null || wheelFR == null || wheelRL == null || wheelRR == null)
			{
			Debug.LogError("Missing VPWheelCollider");
			return;
			}

		// Configure mandatory data per wheel

		ConfigureWheelData(wheelState[0], wheels[0], wheelFL, true);
		ConfigureWheelData(wheelState[1], wheels[1], wheelFR, true);
		ConfigureWheelData(wheelState[2], wheels[2], wheelRL);
		ConfigureWheelData(wheelState[3], wheels[3], wheelRR);

		// Initialize DirectDrive and connect it to the rear wheels via differential

		m_directDrive = new DirectDrive();
		m_differential = new Differential();
		m_differential.settings.gearRatio = 1.0f;

		Block.Connect(wheels[2], 0, m_differential, 0);
		Block.Connect(wheels[3], 0, m_differential, 1);
		Block.Connect(m_differential, 0, m_directDrive, 0);
		}

	// WheelState and Wheel objects must be initialized with a minimum data per wheel

	void ConfigureWheelData (WheelState ws, Wheel wheel, VPWheelCollider wheelCol, bool steerable = false)
		{
		ws.wheelCol = wheelCol;
		ws.steerable = steerable;
		wheel.tireFriction = tireFriction;
		wheel.radius = wheelCol.radius;
		wheel.mass = wheelCol.mass;
		}

	// Steering is done separately for allowing smooth movement of the wheels
	// even in slow motion.

	protected override void DoSteer ()
		{
		float angle = steerInput * maxSteerAngle;
		wheelState[0].steerAngle = angle;
		wheelState[1].steerAngle = angle;
		}

	// Set up the state values at the blocks, tires, etc.

	protected override void DoUpdateBlocks ()
		{
		// Feed the DirectDrive with the values and input from the controller's properties

		m_directDrive.motorInput = driveInput;
		m_directDrive.maxMotorTorque = maxDriveTorque;
		m_directDrive.maxRpm = maxDriveRpm;

		// Set the brakes at the Wheel blocks

		float brakeTorque = brakeInput * maxBrakeTorque;
		wheels[0].Tbrake = brakeTorque;
		wheels[1].Tbrake = brakeTorque;
		wheels[2].Tbrake = brakeTorque;
		wheels[3].Tbrake = brakeTorque;
		}
	}
```

**SimpleVehicleControllerInput.cs**
```
using UnityEngine;

[RequireComponent(typeof(SimpleVehicleController))]
public class SimpleVehicleControllerInput : MonoBehaviour
	{
	SimpleVehicleController m_vehicle;

	void OnEnable ()
		{
		m_vehicle = GetComponent<SimpleVehicleController>();
		}

	void Update ()
		{
		float steerInput = Mathf.Clamp(Input.GetAxis("Horizontal"), -1.0f, 1.0f);

		float throttleAndBrakeAxisValue = Input.GetAxis("Vertical");
		float throttleInput = Mathf.Clamp01(throttleAndBrakeAxisValue);
		float brakeInput = Mathf.Clamp01(-throttleAndBrakeAxisValue);

		if (Input.GetKey(KeyCode.LeftControl) || Input.GetKey(KeyCode.RightControl))
			{
			throttleInput = -brakeInput;
			brakeInput = 0.0f;
			}

		m_vehicle.steerInput = steerInput;
		m_vehicle.driveInput = throttleInput;
		m_vehicle.brakeInput = brakeInput;
		}
	}

```