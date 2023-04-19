<script lang="ts">
	interface PrimitiveType {
		structType: false;
		size: number;
	}

	type CType = StructDef | PrimitiveType;

	interface StructMemberDef {
		type: number;
		isArray: boolean;
		n: number;
		name: string;
	}

	interface StructDef {
		structType: true;
		name: string;
		members: StructMemberDef[];
	}

	const primitives: PrimitiveType[] = [
		{ structType: false, size: 8 },
		{ structType: false, size: 4 },
		{ structType: false, size: 2 },
		{ structType: false, size: 1 }
	];
	let structs: StructDef[] = [];

	function resolveType(n: number): CType {
		return n < primitives.length ? primitives[n] : structs[n - primitives.length];
	}

	function addStruct() {
		structs = [...structs, { structType: true, name: '', members: [] }];
	}

	function addMember(struct: StructDef) {
		return () => {
			structs = structs.map((s) =>
				s !== struct
					? s
					: { ...s, members: [...s.members, { name: '', type: 0, isArray: false, n: 1 }] }
			);
		};
	}

	function removeMember(struct: StructDef, member: StructMemberDef) {
		return () => {
			structs = structs.map((s) =>
				s != struct ? s : { ...s, members: s.members.filter((m) => m !== member) }
			);
		};
	}

	function getStructAlignment(struct: StructDef): number {
		return struct.members
			.map((m) => {
				const type = resolveType(m.type);
				return type.structType ? getStructAlignment(type) : type.size;
			})
			.reduce((a, b) => Math.max(a, b), 0);
	}

	function getStructSize(struct: StructDef): number {
		let offset = 0;
		for (const member of struct.members) {
			const type = resolveType(member.type);
			if (type.structType) {
				const alignment = getStructAlignment(type);
				if (offset % alignment !== 0) {
					offset += alignment - (offset % alignment);
				}
				offset += getStructSize(type);
			} else {
				if (offset % type.size !== 0) {
					offset += type.size - (offset % type.size);
				}
				offset += type.size;
			}
		}

		const alignment = getStructAlignment(struct);
		return Math.ceil(offset / alignment) * alignment;
	}

	interface StructLayoutRecord {
		label: string;
		offset: number;
		size: number;
	}

	function getStructLayout(
		struct: StructDef,
		offset: number = 0,
		parent: string = ''
	): StructLayoutRecord[] {
		let layout: StructLayoutRecord[] = [];
		const addMember = (member: StructMemberDef, name: string) => {
			const type = resolveType(member.type);
			if (type.structType) {
				const alignment = getStructAlignment(type);

				if (offset % alignment !== 0) {
					layout.push({ label: 'padding', offset, size: alignment - (offset % alignment) });
					offset += alignment - (offset % alignment);
				}

				layout.push({ label: `${parent}${name}`, offset, size: getStructSize(type) });
				layout.push(...getStructLayout(type, offset, `${parent}${member.name}.`));
				offset += getStructSize(type);
			} else {
				if (offset % type.size !== 0) {
					layout.push({ label: 'padding', offset, size: type.size - (offset % type.size) });
					offset += type.size - (offset % type.size);
				}

				layout.push({ label: `${parent}${name}`, offset, size: type.size });
				offset += type.size;
			}
		};

		for (let i = 0; i < struct.members.length; i++) {
			const member = struct.members[i];

			if (member.isArray)
				for (let j = 0; j < member.n; j++) {
					addMember(member, `${member.name}[${j}]`);
				}
			else addMember(member, member.name);
		}

		const alignment = getStructAlignment(struct);
		if (offset % alignment !== 0) {
			layout.push({ label: 'padding', offset, size: alignment - (offset % alignment) });
		}

		return layout;
	}
</script>

<div class="container">
	<div class="row">
		<div class="col-6 offset-3 my-4">
			<div class="card">
				<div class="card-body">
					<h1>Struct Layout Tool</h1>

					<div class="alert alert-primary">
						<strong>Note: </strong> An array of unspecified size should be treated like a pointer, not
						an array. Pointers are primitives the size of the addresses on the system.
					</div>

					{#each structs as struct, idx}
						<div class="card my-4">
							<div class="card-body">
								<div class="mb-3">
									<label for={`struct-name-${idx}`} class="form-label">Struct Name</label>
									<input
										type="text"
										class="form-control"
										id={`struct-name-${idx}`}
										bind:value={struct.name}
									/>
								</div>

								{#each struct.members as member, mIdx}
									<div class="card my-4">
										<div class="card-body">
											<div class="mb-3">
												<label for={`member-name-${idx}-${mIdx}`} class="form-label"
													>Member Name</label
												>
												<input
													type="text"
													class="form-control"
													id={`member-name-${idx}-${mIdx}`}
													bind:value={member.name}
												/>
											</div>
											<div class="mb-3">
												<label for={`member-type-${idx}-${mIdx}`} class="form-label"
													>Member Type</label
												>
												<select
													class="form-select"
													id={`member-type-${idx}-${mIdx}`}
													bind:value={member.type}
												>
													{#each primitives as primitive, pidx}
														<option value={pidx}>Primitve - {primitive.size} byte</option>
													{/each}
													{#each structs as iter_struct, sidx}
														{#if iter_struct !== struct}
															<option value={sidx + primitives.length}>{iter_struct.name}</option>
														{/if}
													{/each}
												</select>
											</div>
											<div class="mb-3">
												<input
													class="form-check-input"
													type="checkbox"
													id={`member-is-array-${idx}-${mIdx}`}
													bind:checked={member.isArray}
												/>
												<label for={`member-is-array-${idx}-${mIdx}`} class="form-check-label"
													>Is Array?</label
												>
											</div>
											{#if member.isArray}
												<div class="mb-3">
													<label for={`member-n-${idx}-${mIdx}`} class="form-label"
														>Array Length</label
													>
													<input
														type="number"
														class="form-control"
														id={`member-n-${idx}-${mIdx}`}
														bind:value={member.n}
													/>
												</div>
											{/if}

											<button class="btn btn-danger" on:click={removeMember(struct, member)}
												>Remove</button
											>
										</div>
									</div>
								{/each}

								<button class="btn btn-primary" on:click={addMember(struct)}>+ Member</button>
							</div>
						</div>
					{/each}

					<button class="btn btn-primary" on:click={addStruct}>+ Struct</button>

					{#if structs.filter((s) => getStructSize(s)).length !== 0}
						<div class="card my-4">
							<div class="card-body">
								<h2>Structs</h2>

								{#each structs as struct}
									{#if getStructSize(struct)}
										<div class="card my-4">
											<div class="card-body">
												<h3>{struct.name}</h3>
												<p><strong>Size: </strong> {getStructSize(struct)}</p>
												<p><strong>Alignment: </strong> {getStructAlignment(struct)}</p>

												<table class="table table-striped">
													<thead>
														<tr>
															<th scope="col">Offset</th>
															<th scope="col">Label</th>
															<th scope="col">Size</th>
														</tr>
													</thead>
													<tbody>
														{#each getStructLayout(struct) as layoutRecord}
															<tr>
																<td>0x{layoutRecord.offset.toString(16)}</td>
																<td>{layoutRecord.label}</td>
																<td>0x{layoutRecord.size.toString(16)}</td>
															</tr>
														{/each}
													</tbody>
												</table>
											</div>
										</div>
									{/if}
								{/each}
							</div>
						</div>
					{/if}
				</div>
			</div>
		</div>
	</div>
</div>
