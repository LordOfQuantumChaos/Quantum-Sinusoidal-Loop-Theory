# Quantum-Sinusoidal-Loop-Theory
Simulation code for wave trapping, energy harvest, and quantum theory details
import numpy as np
import matplotlib.pyplot as plt
from scipy.fft import fftn, ifftn
from scipy.optimize import curve_fit

# Function for KG plate-gap sim + harvest
def run_kg_sim(d):
    L = 10.0
    N = 64  # Demo; scale to 128 for precision
    dx = L / N
    dt = 0.0005
    num_steps = 400
    m = 0.5
    V0 = 1.0
    sigma = 0.5
    noise_level = 0.2
    eta = 0.8

    x = np.linspace(0, L, N)
    y = np.linspace(0, L, N)
    z = np.linspace(0, L, N)
    X, Y, Z = np.meshgrid(x, y, z, indexing='ij')

    phi = [0, 2*np.pi/3, 4*np.pi/3]

    V = np.zeros((N, N, N))
    for i in range(3):
        coord = [X, Y, Z][i]
        V += 0.1 * V0 * np.sin(4 * np.pi * (coord + phi[i]))**2

    barb_positions = np.array([
        [L/2, L/2, d/2], [L/2 + 1, L/2, d/2], [L/2 - 1, L/2, d/2],
        [L/2, L/2 + 1, d/2], [L/2, L/2 - 1, d/2],
        [L/2, L/2, d/2 + 0.5], [L/2, L/2, d/2 - 0.5],
        [L/2 + 0.5, L/2 + 0.5, d/2 + 0.5]
    ])
    A_j = np.ones(8) + np.random.uniform(-noise_level, noise_level, 8)
    for j in range(8):
        r_j = barb_positions[j]
        V += A_j[j] * np.exp(- ((X - r_j[0])**2 + (Y - r_j[1])**2 + (Z - r_j[2])**2) / (2 * sigma**2))

    V[Z < 0] = 1e10
    V[Z > d] = 1e10

    sigma_wave = 1.0
    r0 = np.array([L/2, L/2, d/2])
    k0 = np.array([0, 0, 5])
    psi = np.exp(- ((X - r0[0])**2 + (Y - r0[1])**2 + (Z - r0[2])**2) / (4 * sigma_wave**2)) * np.exp(1j * (k0[0]*X + k0[1]*Y + k0[2]*Z))
    psi += np.random.uniform(-noise_level, noise_level, psi.shape)
    psi = psi / np.sqrt(np.sum(np.abs(psi)**2) * dx**3)
    dpsi_dt = np.zeros_like(psi)

    kx = 2 * np.pi * np.fft.fftfreq(N, dx)
    ky = 2 * np.pi * np.fft.fftfreq(N, dx)
    kz = 2 * np.pi * np.fft.fftfreq(N, dx)
    KX, KY, KZ = np.meshgrid(kx, ky, kz, indexing='ij')
    K2 = KX**2 + KY**2 + KZ**2

    m2_V = m**2 + V

    energy_harvest = 0.0
    total_energy = 0.0

    for step in range(num_steps):
        psi += dt / 2 * dpsi_dt
        psi_k = fftn(psi)
        psi_k *= np.exp(-1j * dt * np.sqrt(K2 + m**2))
        psi = ifftn(psi_k)
        dpsi_dt -= dt * m2_V * psi
        psi += dt / 2 * dpsi_dt
        energy_harvest += eta * dt * np.sum(np.abs(psi)**2 * V * dx**3)
        total_energy += np.sum(np.abs(psi)**2 * dx**3)

    avg_energy = total_energy / num_steps

    initial_density = np.sum(np.abs(psi[:, :, 0])**2 * dx**2)
    final_density = np.sum(np.abs(psi[:, :, int(d/dx)])**2 * dx**2)
    transmission = final_density / initial_density if initial_density > 0 else 0

    force_proxy = np.gradient([total_energy for _ in range(num_steps)])[-1] / d  # Simplified proxy

    return transmission, energy_harvest, energy_harvest / (num_steps * dt * L**2), avg_energy, force_proxy

# Run for multiple d
d_values = np.linspace(1.0, 5.0, 5)
transmissions = []
harvests = []
powers = []
energies = []
forces = []

for d in d_values:
    trans, harvest, power, energy, force = run_kg_sim(d)
    transmissions.append(trans)
    harvests.append(harvest)
    powers.append(power)
    energies.append(energy)
    forces.append(force)

# Force fit to 1/d^4
d_centers = (d_values[:-1] + d_values[1:]) / 2
force_proxy = np.diff(energies) / np.diff(d_values)

def casimir_fit(d, a):
    return a / d**4

popt, _ = curve_fit(casimir_fit, d_centers, np.abs(force_proxy))

# Outputs
print("d Values:", d_values)
print("Transmissions:", transmissions)
print("Harvested Energies (J):", harvests)
print("Power Densities (W/m²):", powers)
print("Average Energies:", energies)
print("Force Proxies (dE/dd):", force_proxy)
print("Casimir Fit Parameter a:", popt[0])

# QED Lamb/g-2 Proxy (H Atom with V(r) Pert)
atom = 'H 0 0 0'
basis = 'cc-pVQZ'
mol = gto.M(atom=atom, basis=basis)

mf = scf.RHF(mol).run()

mp2 = mp.MP2(mf).run()

ccsd = cc.CCSD(mf).run()

density = mf.make_rdm1()
r_grid = np.linspace(0, 10, 1000)
pert_shift = np.trapz(density[0,0] * (0.1 * V0 * np.sin(4 * np.pi * r_grid)**2 + np.exp(-r_grid**2 / (2 * sigma**2))), r_grid)

alpha = -ccsd.polarizability()[0,0]
g2_proxy = alpha / (4 * mol.nelectron)

lamb_known = 1057.845
g2_known = 0.00115965218073

print(f"HF Energy: {mf.e_tot:.6f} Ha")
print(f"MP2 Corr: {mp2.e_corr:.6f} Ha")
print(f"CCSD Energy: {ccsd.e_tot:.6f} Ha")
print(f"Pert Shift from V(r): {pert_shift:.6e} Ha")
print(f"Known Lamb: {lamb_known} MHz")
print(f"g-2 Proxy: {g2_proxy:.6e}")
print(f"Known g-2: {g2_known}")

dev_lamb = abs(pert_shift - 4.37e-6) / 4.37e-6 * 100
print(f"Lamb Deviation: {dev_lamb:.2f}%")

# Plot Force vs d (loglog)
plt.loglog(d_centers, np.abs(force_proxy), 'o', label='Sim Data')
d_fit = np.linspace(min(d_values), max(d_values), 100)
plt.loglog(d_fit, casimir_fit(d_fit, *popt), '-', label='1/d^4 Fit')
plt.xlabel('Plate Gap d')
plt.ylabel('Force Proxy |dE/dd|')
plt.title('Casimir-Like Force in Plate-Gap Config')
plt.legend()
plt.show()
</parameter>
</xai:function_call>
