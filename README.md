
Đây là code gốc cho video hướng dẫn làm flight simulator của mình. Subscribe for me on Youtube. Search @VNeseIT, there's a lot of cool things in there!

# Code-for-Mig-21
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class AirplaneController : MonoBehaviour
{
    [SerializeField] List<AeroSurface> controlSurfaces = null;
    [SerializeField] List<WheelCollider> wheels = null;
    [SerializeField] float rollControlSensitivity = 0.35f;
    [SerializeField] float pitchControlSensitivity = 0.02f;
    [SerializeField] float yawControlSensitivity = 0.2f;

    [Range(-1, 1)] public float Pitch;
    [Range(-1, 1)] public float Yaw;
    [Range(-1, 1)] public float Roll;
    [Range(0, 1)] public float Flap;
    [SerializeField] Text displayText = null;

    float thrustPercent;
    float brakesTorque;

    AircraftPhysics aircraftPhysics;
    Rigidbody rb;

    public float throttleIncrement = 0.01f;
    public float maxThrust = 50000f;
    public float liftCoefficient = 1.5f;
    public float dragCoefficient = 0.02f;
    public float downwardForceFactor = 2f;
    public float maxSpeed = 2230f; // Mig-21's maximum speed

    private void Start()
    {
        aircraftPhysics = GetComponent<AircraftPhysics>();
        rb = GetComponent<Rigidbody>();
        rb.mass = 2125f;
        rb.drag = 0.02f;
        rb.angularDrag = 0.02f;
    }

    private void Update()
    {
        Pitch = Input.GetAxis("Vertical");
        Roll = Input.GetAxis("Horizontal");
        Yaw = Input.GetAxis("Yaw");

        if (Input.GetKey(KeyCode.PageUp))
        {
            thrustPercent = Mathf.Clamp(thrustPercent + throttleIncrement, 0, 1);
        }
        if (Input.GetKey(KeyCode.PageDown))
        {
            thrustPercent = Mathf.Clamp(thrustPercent - throttleIncrement, 0, 1);
        }

        if (Input.GetKeyDown(KeyCode.F))
        {
            Flap = Flap > 0 ? 0 : 0.15f;
        }

        if (Input.GetKeyDown(KeyCode.B))
        {
            brakesTorque = brakesTorque > 0 ? 0 : 2300f;
        }

        // Changing the display a little bit...
        float rawSpeed = rb.velocity.magnitude * 3.6f; // Convert from m/s to km/h
        float displaySpeed = Mathf.Min(rawSpeed/3, maxSpeed); // Maxmimum speed to display
        float altidude = transform.position.y + 1f;
        displayText.text = "V: " + ((int)displaySpeed).ToString("D4") + " km/h\n";
        displayText.text += "A: " + ((int)altidude).ToString("D4") + " m\n";
        displayText.text += "T: " + (int)(thrustPercent * 100) + "%\n";
        displayText.text += brakesTorque > 0 ? "B: ON" : "B: OFF";

        // Changing the pitching angle
        if (Pitch != 0)
        {
            float pitchChange = Pitch * pitchControlSensitivity * Time.deltaTime;
            transform.Rotate(Vector3.right, pitchChange, Space.Self);
        }
    }

    private void FixedUpdate()
    {
        SetControlSurfacesAngles(Pitch, Roll, Yaw, Flap);
        ApplyAerodynamicForces();

        foreach (var wheel in wheels)
        {
            wheel.brakeTorque = brakesTorque;
            wheel.motorTorque = 0.01f;
        }
    }

    private void ApplyAerodynamicForces()
    {
        // Calculating the lifting direction based on its angle
        Vector3 thrustDirection = transform.forward;
        Vector3 thrustForce = thrustDirection * thrustPercent * maxThrust;
        rb.AddForce(thrustForce);

        // Calculating the draging force
        float speed = rb.velocity.magnitude;
        Vector3 dragForce = -rb.velocity.normalized * dragCoefficient * speed * speed;
        rb.AddForce(dragForce);

        // Calculating the lifting force
        Vector3 liftDirection = Vector3.ProjectOnPlane(transform.up, rb.velocity.normalized);
        float angleOfAttack = Vector3.Angle(transform.forward, rb.velocity) * Mathf.Deg2Rad;
        float liftCoefficient = Mathf.Sin(2 * angleOfAttack) * this.liftCoefficient;
        float liftForceMagnitude = 0.5f * liftCoefficient * speed * speed;
        Vector3 liftForce = liftDirection.normalized * liftForceMagnitude;

        rb.AddForce(liftForce);

        // Increase the pitching when the pitch is negative
        if (Pitch < 0)
        {
            float downwardForceMagnitude = -Pitch * pitchControlSensitivity * speed * speed * downwardForceFactor;
            Vector3 downwardForce = transform.forward * downwardForceMagnitude;
            rb.AddForce(downwardForce);

            // Adding gravity...
            Vector3 gravitationalForce = Vector3.down * Mathf.Abs(Pitch) * rb.mass * Physics.gravity.magnitude * 0.5f;
            rb.AddForce(gravitationalForce);
        }
    }

    public void SetControlSurfacesAngles(float pitch, float roll, float yaw, float flap)
    {
        foreach (var surface in controlSurfaces)
        {
            if (surface == null || !surface.IsControlSurface) continue;
            switch (surface.InputType)
            {
                case ControlInputType.Pitch:
                    surface.SetFlapAngle(pitch * pitchControlSensitivity / 2.5f * surface.InputMultiplyer);
                    break;
                case ControlInputType.Roll:
                    surface.SetFlapAngle(roll * rollControlSensitivity * surface.InputMultiplyer);
                    break;
                case ControlInputType.Yaw:
                    surface.SetFlapAngle(yaw * yawControlSensitivity * surface.InputMultiplyer);
                    break;
                case ControlInputType.Flap:
                    surface.SetFlapAngle(Flap * surface.InputMultiplyer);
                    break;
            }
        }
    }

    private void OnDrawGizmos()
    {
        if (!Application.isPlaying)
            SetControlSurfacesAngles(Pitch, Roll, Yaw, Flap);
    }
}
// Done!
